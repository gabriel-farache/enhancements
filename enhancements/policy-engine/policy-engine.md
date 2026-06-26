---
title: Policy API & Execution Engine
authors:
  - "@ygalblum"
reviewers:
  - "@gciavarrini"
  - "@machacekondra"
  - "@jenniferubah"
  - "@croadfel"
  - "@flocati"
  - "@pkliczewski"
  - "@gabriel-farache"
approvers:
  - TBD
creation-date: 2025-12-15
see-also:
  - "/enhancements/environment-agent/environment-agent.md"
---

# Policy API & Execution Engine

## Summary

This enhancement defines the Management and Execution API and Workflow of the
DCM Policy Engine.

With the introduction of the Environment Agent layer, the Policy Engine selects
an Agent (rather than a Service Provider) to handle the request, and can
constrain selection by environment.

## Motivation

The Policy Engine operates as a specialized microservice within the Data Center
Management (DCM) application responsible for governing service creation and
modification (e.g., VirtualMachines, Containers). It enables Admins,
Tenant-Admins, and Users to inject logic that validates (Approve/Reject),
mutates (Defaulting/Altering) and assigns Agents to request payloads using an
embedded [Open Policy Agent (OPA)](https://www.openpolicyagent.org/docs) engine
and [Rego](https://www.openpolicyagent.org/docs/policy-language).

OPA is embedded as a Go library within the Policy Engine process rather than
deployed as a separate sidecar service. Rego source code is persisted in the
database alongside policy metadata, ensuring policies survive service restarts.
On startup, the engine recompiles all stored policies from the database,
eliminating the persistence gap inherent in an external OPA deployment.

### Goal

Define the flow of how Policies are managed and used by the Policy Engine

- Define Policy types - Global, Tenant, User
- Define Policy management
  - How policies should be added/updated
  - How policies will be stored
- Define Policy execution
  - Policy priority
  - Value immutability and constraints
- Determine the enforcement engine and policy language
  - For V1, OPA (embedded as a Go library) and Rego will be used
  - Other alternatives may be considered in future versions
- Define the input format
- Define the output format

### Non-Goals

- Policy implementation
- Actionable OpenAPI specification
- While this ADR references `Tenant` level `Policy` and `ID`, `Tenants` are not
  supported in V1

## Core Concepts & Definitions

### Policy Responsibilities

Every policy may return one or more of the following outputs

1. **Reject:** Requests are approved by default. Policies may decide whether the
   request should be _Rejected_.
2. **Mutation:** Modifying the request payload (e.g., injecting default labels)
   by providing a patch map.
3. **Field Constraints:** Defining the mutability of fields for _subsequent_
   policies in the chain.
4. **Agent Selection:** Policies may set a target agent and/or agent constraints
   (including environment constraints)

### Policy Scope & Hierarchy (Execution Order)

The execution order is strictly determined by **Level** first, then
**Priority**.

1. **Global:** (Super Admin) - Runs first.
2. **Tenant:** (Tenant Admin) - Runs second.
3. **User:** (End User) - Runs last.

_Within each level, policies are sorted by priority: lower integers indicate
higher priority._

### The "Rego Contract"

#### Input

The input payload includes:

- `spec` - The current patched request payload
  - Assumption - While policies do not have to be specific for Service Types
    they will need to know the expected content
- `constraints` - The current constraints context (accumulated from prior
  policies)
- `agent` - The currently selected agent (empty string initially, populated as
  policies are evaluated)
- `agent_constraints` - The current agent constraints (accumulated from prior
  policies)
- `available_agents` - List of eligible agents with metadata
  `[{name, environment, serviceTypes, cost}]`, provided by Placement Manager
- `exclude_agents` - List of agent names to exclude from selection (used during
  re-evaluation after queued timeout)

#### Output

Following the policy responsibilities, the output should be comprised of the
following elements

- **rejected** (bool) - since requests are approved by default, policies may
  reject them.
- **rejection_reason** (string, optional) - reason for rejection
- **selected_agent** (string, optional) - the name of the agent chosen to handle
  the request
- **agent_constraints** (object, optional) -
  - `allow_list` - list of allowed agent names
  - `patterns` - list of regex patterns for matching allowed agents
  - `environment_constraints` - environment-level constraints
    - `allow_list` - list of allowed environment identifiers
    - `patterns` - list of regex patterns for matching allowed environments
- **patch** (map, optional) - a dictionary of the corresponding service type for
  setting values. Each internal key is optional
- **constraints** (map, optional) - follows
  [JSON Schema (draft 2020-12)](https://json-schema.org/draft/2020-12/json-schema-validation).

  This standard supports:
  - Immutable: _const_
  - Numeric constraints: _minimum, maximum, multipleOf_
  - String patterns: _pattern, minLength, maxLength_
  - Enumerations: _enum_
  - Array constraints: _minItems, maxItems_
  - Conditional logic: _if/then/else_

  For the complete validation vocabulary, see the
  [JSON Schema Validation specification](https://json-schema.org/draft/2020-12/json-schema-validation).

### Policy Code Ownership and Responsibilities

- DCM Admins, Tenant-Admins and Users implement the policies' REGO code
- DCM Admins, Tenant-Admins and Users are responsible for correct registration
  of the policies
- DCM Admins, Tenant-Admins and Users are responsible for the accuracy and
  performance of the policies
- Trying to register a REGO code snippet that fails compilation will fail

## System Architecture

The Policy API serves two distinct functions:

1. Management Plane: CRUD operations for Policy definitions. Rego source code is
   persisted in the database and the embedded OPA engine is recompiled after
   every CRUD mutation.
2. Execution Plane: Service requests evaluation against active policies using
   the embedded OPA engine.

### Policy Management

#### Policy Registration Flow

```mermaid
sequenceDiagram
    participant User
    participant PolicyEngine
    participant Database

    User->>PolicyEngine: POST /api/v1/policies
    PolicyEngine->>Database: Check unique Name and Priority for policy type
    alt Uniqueness check failed
        PolicyEngine-->>User: Error response
    else Uniqueness check passed
        PolicyEngine->>PolicyEngine: Generate UUID
        PolicyEngine->>Database: Store policy metadata and Rego code
        Note right of Database: UUID, Name, RegoCode,<br/>LabelSelector, Policy Type, Priority
        PolicyEngine->>PolicyEngine: Recompile embedded OPA engine
        alt REGO compilation failed
            PolicyEngine->>PolicyEngine: Preserve previous compiled state
            PolicyEngine->>Database: Rollback stored metadata
            PolicyEngine-->>User: Error response
        else REGO compilation succeeded
            PolicyEngine-->>User: Return UUID
        end
    end
```

#### Pseudo API

##### POST /api/v1/policies

###### Payload

- Name
  - Must be unique at its level. That is:
    - All global policies must have unique names
    - All tenant policies must have unique names within their tenant
    - All user policies must have unique names for their user
- Policy Matching Criteria. Treated with AND.
  - Label Selector
- Policy Type
  - Global, Tenant, User
- Priority
  - Must be unique at its level
  - A lower number means a higher priority and therefore will be evaluated first
- REGO Code
- Enabled
  - Optional. Default `true`

###### Response Payload

- Generated UUID

###### Execution Logic & Flow

- Validate the Policy Name and Priority
  - If not unique return an error
- Generate a UUID
- Store the following information in the DB
  - UUID
  - Name
  - Rego Code
  - Policy Type
  - Priority
  - Label Selector
- Recompile the embedded OPA engine with all stored policies
  - The package name is resolved from the compiled AST
  - Compilation is atomic: if it fails, the previous compiled state is preserved
  - If failed, rollback DB and return an error
- Return UUID to caller

##### GET /api/v1/policies

Return the list of policies. Allow for filtering

##### GET /api/v1/policies/{policyId}

Return the specific policy

##### DELETE /api/v1/policies/{policyId}

Delete the specific policy

##### PUT /api/v1/policies/{policyId}

Update the specific policy. Policy name and type are immutable

###### Payload

- Policy Matching Criteria
- Priority
- REGO Code
- Enabled

### Execution Plane

#### Sequence

```mermaid
sequenceDiagram
    participant User
    participant PlacementManager
    participant PolicyEngine
    participant Database

    User->>PlacementManager: Create Service request
    PlacementManager->>PolicyEngine: Validate Payload
    PolicyEngine->>Database: Get matching policies by serviceType and labelSelector
    Database-->>PolicyEngine: List of policies

    loop For each policy
        PolicyEngine->>PolicyEngine: Evaluate policy (embedded OPA)<br/>{spec, agent, constraints, agent_constraints}
        PolicyEngine->>PolicyEngine: Enforce constraints
        PolicyEngine->>PolicyEngine: Mutate payload
        alt Policy rejected or constraint violation
            PolicyEngine-->>PlacementManager: Request rejected
            PlacementManager-->>User: Request rejected
        end
    end

    PolicyEngine-->>PlacementManager: Success with {evaluatedServiceInstance, selectedAgent, status}
    PlacementManager-->>User: Service created
```

#### Pseudo API

##### POST /api/v1alpha1/policies:evaluateRequest

###### Payload

- Service Instance
  - spec - the service specification (flexible schema)
- available_agents - list of agents with metadata (provided by PM)
  `[{name, environment, serviceTypes, cost}]`
- exclude_agents - list of agent names to exclude (optional, used for
  re-evaluation)

###### Execution Logic & Flow

The Engine acts as an orchestrator. It evaluates policies using the embedded OPA
engine, which holds all compiled Rego modules in memory. Evaluation is
concurrent-safe via a read-write lock, allowing policy evaluation to proceed in
parallel with policy management operations.

###### _Pipeline Logic (The "Chain of Responsibility")_

- The Policy API maintains a `ConstraintContext` map in memory for the duration
  of the request.
- Pre-filter: Remove any agents in `exclude_agents` from the `available_agents`
  list before evaluation begins.
- Fetch & Sort:
  - Query DB for enabled policies matching the request payload based on the
    policy's matching criteria.
  - Sort by Level (Global -> Tenant -> User) then Priority (Desc).
- If no policies matching the request payload were found, the request will
  return successfully
- Iterate for each policy P:
  - Evaluate policy using the embedded OPA engine:
    - Invoke the policy's package main rule
    - Pass
      - `spec` - the current patched request payload
      - `agent` - the currently selected agent
      - `constraints` - the accumulated constraint context (if any)
      - `agent_constraints` - the accumulated agent constraints (if any)
      - `available_agents` - the pre-filtered list of eligible agents
      - `exclude_agents` - the list of excluded agent names
  - Check `Reject`
    - If `Reject` is `true`, ABORT IMMEDIATELY (Fail Fast). Return 406.
  - Validate `Constraints`:
    - A lower-level policy cannot "unlock" a field locked by a higher-level
      policy.
    - If it does, ABORT with "Policy Conflict Error"
  - Update `ConstraintContext`:
    - Merge new `Constraints` from Policy P into `ConstraintContext`.
  - Validate `Patch`:
    - Validate `Patch` against `ConstraintContext`.
    - Example: If `ConstraintContext.region` is immutable and Policy P tries to
      patch the `region`, ABORT with "Policy Conflict Error"
  - Apply `Patch`
    - Update service_payload with valid patches.
  - Validate `Agent`
    - If Policy P returned a `selected_agent` and `agent_constraints` exist,
      validate the selected agent against the constraints.
    - If `environment_constraints` exist, validate the selected agent's
      environment against those constraints (Policy Engine uses
      `available_agents` metadata for this).

- Finalize: Return the final payload, selected agent, and status to Placement
  Manager.
  - Response: `{evaluatedServiceInstance, selectedAgent, status}`
  - Status is `APPROVED` if the payload was not modified, `MODIFIED` if any
    patches were applied.

###### _Constraint Validation Example_

- Step 1 (Global Policy):
  - Patch: {"billing_tag": "engineering"}
  - Constraint: {"billing_tag": {"mode": "immutable"}}
  - Result: Payload has billing_tag. Context has billing_tag=immutable.
- Step 2 (User Policy):
  - Patch: {"billing_tag": "marketing"}
  - Action: Engine checks Context. billing_tag is immutable.
- Result: Error. The User policy violates the Global constraint.

###### _Agent/Environment Constraint Validation Example_

- Step 1 (Global Policy):
  - agent_constraints: {environment_constraints: {allow_list: ["prod-eu-west-1",
    "prod-us-east-1"]}}
  - Result: Only agents in prod-eu-west-1 or prod-us-east-1 are eligible
- Step 2 (Tenant Policy):
  - selected_agent: "prod-eu-agent"
  - Validation: Agent's environment is "prod-eu-west-1" (looked up from
    available_agents metadata) — matches allow_list. Valid.
- Step 3 (User Policy):
  - selected_agent: "dev-agent"
  - Validation: Agent's environment is "dev" (looked up from available_agents
    metadata) — NOT in allow_list. Error: violates Global constraint.

## Next Steps

- Cost-based agent selection within agent_constraints
- Resource capacity constraints (totalCpu, totalMemory)
- SP-level constraints passed through to agents
