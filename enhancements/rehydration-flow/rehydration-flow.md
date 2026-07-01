---
title: rehydration-flow
authors:
  - "@ygalblum"
reviewers:
  - "@gciavarrini"
  - "@machacekondra"
  - "@croadfel"
  - "@flocati"
  - "@pkliczewski"
  - "@gabriel-farache"
  - "@jenniferubah"
creation-date: 2026-03-23
see-also:
  - "/enhancements/placement-manager/placement-manager.md"
  - "/enhancements/sp-resource-manager/sp-resource-manager.md"
  - "/enhancements/user-flows/user-flows.md"
  - "/enhancements/environment-agent/environment-agent.md"
---

# Rehydration Flow

## Summary

Rehydration is the process of recreating an existing resource from its original
request (intent). The flow re-evaluates policies against the stored intent and
creates a new resource before deleting the old one. This allows the system to
absorb changes in policies and environment that occurred since the original
resource was provisioned.

## Motivation

Over time, policies, Agent availability, and environment configurations may
change. A resource that was provisioned under a previous set of policies may no
longer comply with current rules, or a more suitable Agent may have become
available. Rehydration enables administrators and users to bring existing
resources in line with the current state of the system without requiring manual
recreation.

### Goals

- Define the end-to-end rehydration flow across Catalog Manager, Placement
  Manager, and SP Resource Manager
- Define new API endpoints for triggering rehydration
- Define how deletion failures are handled when the original Agent is
  unavailable
- Define the deferred cleanup mechanism for resources that could not be deleted

### Non-Goals

- Modifying the original CatalogItemInstance, ServiceType, or CatalogItem
  definitions as part of rehydration
- Supporting partial rehydration (e.g., updating policies without recreating the
  resource)
- Defining update-in-place semantics

## Proposal

### Overview

Rehydration is triggered on an existing CatalogItemInstance. The flow
intentionally does **not** regenerate the ServiceType payload from the
CatalogItem. Instead, it uses the original intent stored in the Placement DB to
ensure that only policy and environment changes are reflected, not changes to
the underlying ServiceType or CatalogItem definitions.

#### ID Separation

The CatalogItemInstance ID used by the Catalog Manager is separate from the
InstanceID used by the Placement Manager and SP Resource Manager. During the
initial create flow, the Catalog Manager generates a InstanceID and passes it
downstream. The Catalog Manager maintains a mapping between its
CatalogItemInstance ID and the InstanceID. This separation is critical for
rehydration: it allows the Catalog Manager to generate a **new** InstanceID for
the recreated resource while the old InstanceID is still in use, avoiding ID
conflicts in the downstream services.

The high-level flow is:

1. User triggers rehydration on a CatalogItemInstance via the Catalog Manager
2. Catalog Manager generates a new InstanceID and calls the Placement Manager
   rehydrate endpoint with both the current and the new InstanceID
3. Placement Manager retrieves the original intent using the current InstanceID
4. Placement Manager re-evaluates policies against the original intent
5. Placement Manager instructs SP Resource Manager to create the new resource
   using the new InstanceID
6. Once the new resource is provisioned, Placement Manager instructs SP Resource
   Manager to delete the old resource using the old InstanceID

### System Architecture

```mermaid
flowchart TD
    CM["Catalog Manager<br/>Trigger Rehydration"]

    subgraph DCM_Core [" "]
        PM["Placement Manager<br/>Orchestrate Rehydration"]

        PE["Policy Manager<br/>Re-evaluate Policies<br/>Agent Selection"]

        SPRM["SP Resource Manager<br/>Publish to Agent Topic<br/>Deferred Cleanup"]

        PM_DB[("Placement DB<br/>Original Intent<br/>Validated Request")]
    end

    MS["Messaging System<br/>(NATS)"]
    AG["Agent<br/>Routes to SPs"]

    CM --> PM
    PM --> PM_DB
    PM --> PE
    PM --> SPRM
    SPRM --> MS
    MS --> AG
```

### API Endpoints

#### Catalog Manager

| Method | Endpoint                                                         | Description                        |
| ------ | ---------------------------------------------------------------- | ---------------------------------- |
| POST   | /api/v1/catalog-item-instances/{catalogItemInstanceId}:rehydrate | Trigger rehydration of an instance |

**POST /api/v1/catalog-item-instances/{catalogItemInstanceId}:rehydrate**

Triggers rehydration of an existing CatalogItemInstance. The Catalog Manager
does **not** regenerate the ServiceType payload. It generates a new InstanceID
and delegates to the Placement Manager rehydrate endpoint, passing both the
current InstanceID and the new InstanceID.

Response: Returns `202 Accepted` if the rehydration process has started.

#### Placement Manager

| Method | Endpoint                                 | Description                    |
| ------ | ---------------------------------------- | ------------------------------ |
| POST   | /api/v1/resources/{instanceId}:rehydrate | Rehydrate an existing resource |

**POST /api/v1/resources/{instanceId}:rehydrate**

Triggers the rehydration of an existing resource. The Placement Manager
retrieves the original intent from the Placement DB and orchestrates creation of
the new resource followed by deletion of the old one.

Request body:

```json
{
  "newInstanceId": "<new-instance-id>"
}
```

Response: Returns `202 Accepted` if the rehydration process has started.

## Design Details

### Rehydration Flow

```mermaid
sequenceDiagram
    autonumber
    actor User
    participant CM as Catalog Manager
    participant CM_DB as Catalog Manager DB
    participant PM as Placement Manager
    participant DB as Placement DB
    participant PE as Policy Manager
    participant SPRM as SP Resource Manager
    participant MS as Messaging System

    User->>CM: POST /api/v1/catalog-item-instances/{catalogItemInstanceId}:rehydrate
    CM->>CM: Generate newResourceId
    CM->>CM_DB: Update resourceId reference to newResourceId

    CM->>PM: POST /api/v1/resources/{resourceId}:rehydrate<br/>{newResourceId}

    activate PM

    PM->>DB: Retrieve original intent by resourceId
    DB-->>PM: {originalRequest, agentName, oldResourceId}

    PM->>DB: Fetch available agents<br/>(healthy, non-Congested)
    DB-->>PM: available_agents list

    PM->>PE: POST /api/v1alpha1/policies:evaluateRequest<br/>{service_instance: {originalSpec}, available_agents}

    alt Policy rejects
        PE-->>PM: 406 Not Acceptable
        PM->>DB: Update record (policy rejected)
        PM-->>CM: Error (policy rejected)
        CM->>CM_DB: Rollback resourceId to currentResourceId
        CM-->>User: Rehydration failed (policy rejected)

    else Policy approves
        PE-->>PM: 200 OK<br/>{evaluatedServiceInstance, selectedAgent, status}

        PM->>DB: Store validated request with newResourceId<br/>{validatedPayload, new agentName}

        PM->>SPRM: POST /api/v1/service-type-instances<br/>{newResourceId, agentName, serviceType, spec}
        activate SPRM

        alt Agent not found or unhealthy
            SPRM-->>PM: Error response
            PM-->>CM: Error (agent unavailable)
            CM->>CM_DB: Rollback resourceId to currentResourceId
            CM-->>User: Rehydration failed
        else Agent healthy
            SPRM->>MS: PUBLISH CloudEvent<br/>topic: {topicName}<br/>type: dcm.request.create<br/>{newResourceId, serviceType, spec}
            SPRM-->>PM: 202 Accepted<br/>{newResourceId, agentName, status: PENDING}
        end
        deactivate SPRM

        PM->>SPRM: DELETE /api/v1/service-type-instances/{oldResourceId}?deferred=true
        activate SPRM
        SPRM->>SPRM: Record pending cleanup<br/>{oldResourceId, agentName}
        SPRM-->>PM: 200 OK (deletion deferred)
        deactivate SPRM

        PM->>DB: Remove old instance record
        PM-->>CM: 202 Accepted {newInstanceId, status}
        CM-->>User: Rehydration started<br/>{status: PENDING}
    end
    deactivate PM
```

### Flow Description

1. **Rehydration Trigger**
   - User sends a POST request to the Catalog Manager rehydrate endpoint
   - Catalog Manager does **not** regenerate the ServiceType payload from the
     CatalogItem. This ensures that only policy and environment changes are
     applied, not changes to the underlying CatalogItem or ServiceType
   - Catalog Manager reads the current resourceId from its database
   - Catalog Manager generates a new resourceId for the downstream services
   - Catalog Manager updates its database with the new resourceId **before**
     calling Placement Manager (see
     [DB-First Update Order](#catalog-manager-db-first-update-order)). The
     update uses compare-and-swap (CAS): it only succeeds if the resourceId
     still matches the value read earlier, preventing concurrent rehydrates from
     both proceeding
   - Catalog Manager then forwards the request to the Placement Manager
     rehydrate endpoint with the current resourceId (in the URL) and the new
     resourceId (in the request body)

2. **Intent Retrieval**
   - Placement Manager retrieves the original intent (the user's original
     request) from the Placement DB using the current InstanceID
   - The original intent includes the spec, the current agentName, and the old
     InstanceID

3. **Fetch Available Agents**
   - Placement Manager queries the Agent Registry for healthy, non-Congested
     agents that support the requested service type
   - The resulting `available_agents` list is passed to the Policy Manager for
     evaluation

4. **Policy Re-evaluation**
   - Placement Manager sends the original intent to the Policy Manager with
     `available_agents` for evaluation against the current policy set
   - Policy Manager evaluates the request through the full policy chain (Global,
     Tenant, User)
   - If the policy rejects the request, the Placement Manager updates the record
     and returns an error to Catalog Manager, which rolls back its database to
     the original resourceId
   - If the policy approves, the Placement Manager receives the evaluated
     payload and the newly selected Agent (`selectedAgent`)

5. **Resource Creation**
   - Placement Manager stores the new validated request in the Placement DB with
     the new InstanceID and the new `agentName`
   - Placement Manager delegates instance creation to SP Resource Manager with
     the new InstanceID, the new agentName, serviceType, and the evaluated spec
   - Since the new InstanceID is different from the old one, there is no ID
     conflict in SP Resource Manager
   - SP Resource Manager publishes a creation CloudEvent to the agent's
     messaging topic
   - On success, the resource enters `PENDING` state
   - On failure, Catalog Manager rolls back its database to the original
     resourceId

6. **Delete Old Resource**
   - Once the new resource is created, Placement Manager requests SP Resource
     Manager to delete the old resource using the old InstanceID with the
     `deferred` flag set to `true`
   - SP Resource Manager immediately records the instance in the cleanup queue
     for background deletion without publishing to the agent (see
     [Deferred Deletion](#deferred-deletion))
   - SP Resource Manager returns success to allow the flow to continue
   - Placement Manager removes the old instance record from the Placement DB and
     returns success to the Catalog Manager

### Handling Deletion of the Old Resource

#### Deferred Deletion

During rehydration, the deletion request is sent with the `deferred` flag set to
`true`. When the SP Resource Manager receives a deferred deletion request, it
does **not** publish a deletion CloudEvent to the Agent. Instead, it immediately
enqueues the instance for background cleanup:

1. The SP Resource Manager records the pending deletion in a **cleanup queue**
   (persisted in the database) with the following information:
   - `instanceId`: The instance to be deleted
   - `agentName`: The Agent that manages the instance
   - `serviceType`: The type of the service
   - `timestamp`: When the deletion was requested
2. The SP Resource Manager returns success to the Placement Manager, allowing
   the rehydration flow to continue

#### Cleanup Mechanism

The SP Resource Manager runs a background cleanup process that periodically
attempts to complete deferred deletions. The cleanup queue serves two purposes:

1. **Rehydration deferred deletions**: When the Placement Manager sends a
   deletion with `deferred=true`, SPRM enqueues it without publishing to the
   Agent (described above).
2. **SP-unavailable deletion failures**: When a regular deletion is rejected by
   the Agent because the SP became Unavailable, SPRM enqueues it for deferred
   retry rather than marking the resource as failed (see
   [Placement Manager — Service Deletion Flow](../placement-manager/placement-manager.md#service-deletion-flow)).

The scheduler applies the following resolution logic for each pending deletion:

```mermaid
flowchart TD
    A[Cleanup scheduler triggers] --> B[Query cleanup queue<br/>for pending deletions]
    B --> C{Any pending?}
    C -->|No| D[Sleep until next interval]
    C -->|Yes| E[For each pending deletion]
    E --> F[Lookup agent<br/>in Agent Registry]
    F --> G{Agent registered?}
    G -->|No| H[Mark as DELETED<br/>Environment presumed decommissioned]
    G -->|Yes| I{Agent advertises<br/>the service type?}
    I -->|No| J[Skip, retry next cycle]
    I -->|Yes| K[Publish deletion CloudEvent<br/>to agent topic]
    K --> L{Deletion succeeded?}
    L -->|Yes| M[Mark as DELETED]
    L -->|No| N[Mark as DELETED<br/>Original SP context is gone]
    M --> D
    N --> D
    H --> D
    J --> D
```

**Resolution rules:**

- **Agent not registered**: The Agent and its environment are presumed
  decommissioned. The resource is marked as `DELETED` and removed from the
  cleanup queue.
- **Agent registered but service type not advertised**: The SP for the service
  type has not recovered yet. The scheduler skips the entry and retries on the
  next cycle.
- **Agent registered and service type advertised — deletion succeeds**: The SP
  processed the deletion. The resource is marked as `DELETED`.
- **Agent registered and service type advertised — deletion fails**: The service
  type is now served by a different SP that has no knowledge of the resource
  (the original SP's infrastructure is gone). The resource is marked as
  `DELETED` because there is nothing left to clean up from DCM's perspective.

**Cleanup queue record:**

```json
{
  "instanceId": "08aa81d1-a0d2-4d5f-a4df-b80addf07781",
  "agentName": "prod-eu-agent",
  "serviceType": "vm",
  "requestedAt": "2026-03-23T10:00:00Z",
  "retryCount": 0,
  "status": "PENDING",
  "lastAttempt": null
}
```

#### Key Characteristics

- **Non-blocking**: Deferred deletion does not publish to the Agent, so the
  calling flow is never blocked by agent latency or availability
- **Persistent**: The cleanup queue is stored in the database to survive
  restarts
- **Automatic retry**: The cleanup process retries deletions once the Agent
  re-advertises the service type
- **Deterministic resolution**: Every terminal outcome marks the resource as
  `DELETED` — either the deletion succeeds, the original SP context is gone, or
  the environment is decommissioned. No entry remains in the queue indefinitely.
- **Idempotent**: Cleanup deletions are idempotent; repeated attempts to delete
  an already-deleted resource are safe

### Placement Manager Rehydration Flowchart

```mermaid
flowchart TD
    A[Receive rehydrate request<br/>for instanceId with newInstanceId] --> B[Retrieve original intent<br/>from Placement DB]
    B --> C{Intent found?}
    C -->|No| D[Return 404 Not Found]
    C -->|Yes| E[Fetch available agents]
    E --> F[Send original intent to<br/>Policy Manager with available_agents]
    F --> G{Policy approved?}
    G -->|No| H[Update record in Placement DB]
    H --> I[Return error to Catalog Manager]
    G -->|Yes| J[Store validated request<br/>with newInstanceId and agentName]
    J --> K[Forward to SP Resource Manager<br/>with newInstanceId, agentName, and spec]
    K --> L{Creation succeeded?}
    L -->|No| I
    L -->|Yes| M[Request SP Resource Manager<br/>to delete old resource<br/>with deferred flag]
    M --> N[Remove old instance record<br/>from Placement DB]
    N --> O[Return 202 Accepted<br/>to Catalog Manager]
```

### Key Characteristics

- **Intent Preservation**: Rehydration operates on the original user intent, not
  the current CatalogItem or ServiceType definitions. This ensures that only
  policy and environment changes are reflected
- **Create-before-Delete**: The new resource is created before the old one is
  deleted. This ensures the system is never left without a running resource
  during the rehydration process
- **ID Separation**: The CatalogItemInstance ID is separate from the InstanceID
  used downstream. This allows the Catalog Manager to issue a new InstanceID for
  the recreated resource, avoiding ID conflicts in downstream services
- **Policy Re-evaluation**: Every rehydration re-evaluates the full policy
  chain, potentially selecting a different Agent or applying different mutations
- **Deferred Cleanup**: Deletion of the old resource is always deferred during
  rehydration. The SP Resource Manager enqueues the old instance for background
  cleanup without publishing to the Agent, ensuring the rehydration flow is
  never blocked by agent availability or errors
- **Idempotent Rehydration**: Rehydrating an already-rehydrated resource works
  the same way; a new resource is created from the original intent and the
  current resource is deleted afterward

### Catalog Manager: DB-First Update Order

The Catalog Manager uses a **DB-first** approach when updating the `resource_id`
during rehydration. The database is updated before calling Placement Manager,
and rolled back if the PM call fails.

#### Why DB-First?

The alternative approach (PM-first) would call Placement Manager before updating
the database. While this ensures the database always points to a resource that
exists in PM, it introduces a significant risk: **orphaned resources**.

| Approach     | Tradeoff                                                                                                                                                                                                 |
| ------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **PM-first** | DB always points to something real, but if the DB update fails, PM has provisioned a resource that the DB doesn't reference. These orphans are invisible to the system and silently leak infrastructure. |
| **DB-first** | DB may briefly point to a `resource_id` that PM hasn't provisioned yet (a window of milliseconds), but we rollback immediately on PM failure, and any inconsistency is easily detected.                  |

The key insight: **DB inconsistencies are cheap to detect and fix; PM orphans
leak real infrastructure silently.**
