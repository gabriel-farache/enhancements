---
title: catalog-item-schema
authors:
  - "@gciavarrini"
reviewers:
  - "@machacekondra"
  - "@ygalblum"
  - "@jenniferubah"
  - "@croadfel"
  - "@flocati"
  - "@pkliczewski"
  - "@gabriel-farache"
approvers:
  - "@machacekondra"
  - "@ygalblum"
  - "@jenniferubah"
  - "@flocati"
  - "@gabriel-farache"
creation-date: 2025-12-05
see-also:
  - "/enhancements/service-type-definitions/service-type-definitions.md"
  - "/enhancements/declarative-api/declarative-api.md"
replaces:
  - TBD
superseded-by:
  - TBD
---

# Catalog Item Schema

## Summary

This ADR defines the structure for Catalog Items (curated offerings) and Catalog
Item Instances (user orders) in the DCM Service Catalog.

## Motivation

Catalog Items wrap service specifications with validation rules and defaults,
enabling administrators to create curated offerings. The design is both
provider-agnostic and service type-agnostic. Catalog Items work with any service
type defined in
[Service Type Definition](https://raw.githubusercontent.com/dcm-project/enhancements/main/enhancements/service-type-definitions/service-type-definitions.md).
Every catalog item is a blueprint of one or more named resources. Each resource
declares its`serviceType` from the registry, optional `requiresResources`, and
field configurations.

### Goals

- Define the structure for Catalog Items and Catalog Item Instances
- Maintain independence between service types while applying consistent design
  patterns
- Add validation rule
- Enable administrators to create curated offerings for users
- Design catalog schemas as provider-agnostic specifications that Service
  Providers translate to their native platform formats
- Define catalog items for multi-resource stacks (n-tier applications) using the
  same `resources[]` shape as single-resource offerings

### Non-Goals

- Defining the service schemas themselves, see
  [Service Type Definition](https://raw.githubusercontent.com/dcm-project/enhancements/main/enhancements/service-type-definitions/service-type-definitions.md)

## Proposal

The catalog schema acts as a _translation layer_ between what the DCM users want
(abstract service specifications) and what providers deliver (platform-specific
implementations).

### Implementation Details/Notes/Constraints

#### Catalog item blueprint

Every catalog item defines `spec.resources` (min 1). Each entry is a named
resource with its own `serviceType`, optional `requiresResources`, and `fields`
for defaults and governance.

| Kind            | `resources` length | Provisioning                                          |
| :-------------- | :----------------- | :---------------------------------------------------- |
| Single-resource | 1                  | One graph node                                        |
| Multi-resource  | > 1                | One graph node per entry; DAG via `requiresResources` |

Orchestration (DAG sort, per-node policy, per-level create) is defined in
[Declarative API](/enhancements/declarative-api/declarative-api.md).

##### CatalogItem spec

| Field      | Required | Type   | Description                                   |
| :--------- | :------- | :----- | :-------------------------------------------- |
| apiVersion | Yes      | string | CatalogItem schema version (e.g., _v1alpha1_) |
| resources  | Yes      | array  | Blueprint resources (min 1); see below        |

Each entry in `resources`:

| Field             | Required | Type   | Description                                                               |
| :---------------- | :------- | :----- | :------------------------------------------------------------------------ |
| name              | Yes      | string | Stable identifier within the blueprint (e.g., _main_, _ordersDb_)         |
| serviceType       | Yes      | string | Service type from the registry (_vm_, _container_, _database_, _cluster_) |
| requiresResources | No       | array  | Other blueprint `name` values that must reach Ready before this resource  |
| fields            | Yes      | array  | Defaults and validation for this resource (see below)                     |

##### Example: Production Postgres (single resource)

```yaml
apiVersion: v1alpha1
kind: CatalogItem
metadata:
  name: production-postgres
spec:
  resources:
    - name: prod-db
      serviceType: database
      fields:
        - path: "engine"
          default: "postgresql"
        - path: "version"
          editable: true
          default: "15"
          validationSchema: { enum: ["14", "15", "16"] }
        - path: "resources.cpu"
          editable: true
          default: 4
          validationSchema: { minimum: 2, maximum: 16 }
        - path: "resources.memory"
          editable: true
          default: "16GB"
```

##### Example: Development VM (single resource)

```yaml
apiVersion: v1alpha1
kind: CatalogItem
metadata:
  name: development-vm
  displayName: "Development VM"
spec:
  resources:
    - name: dev-vm
      serviceType: vm
      fields:
        - path: "vcpu.count"
          displayName: "CPU Count"
          editable: true
          default: 2
          validationSchema: { minimum: 1, maximum: 4 }
        - path: "memory.size"
          displayName: "Memory"
          editable: true
          default: "4GB"
          validationSchema: { minimum: 2, maximum: 8 }
        - path: "guestOS.type"
          displayName: "Operating System"
          editable: false
          default: "rhel-9"
```

Multiple catalog items can reference the same `serviceType` with different
`validationSchema` constraints: a `Production VM` item could require
`vcpu.count` between 4-16 instead of 1-4, while sharing the same underlying `vm`
ServiceType definition.

| Field            | Required | Type    | Default | Description                                                                   |
| :--------------- | :------- | :------ | :------ | :---------------------------------------------------------------------------- |
| path             | Yes      | string  | -       | Field path in service schema (e.g., _vcpu.count_)                             |
| displayName      | No       | string  | -       | Human-readable label for UI. If not set, derived from the path                |
| editable         | No       | boolean | false   | Whether users can modify this field                                           |
| default          | No       | any     | -       | Default value for this field                                                  |
| validationSchema | No       | object  | -       | JSON Schema rules (only applies if editable)                                  |
| dependsOn        | No       | object  | -       | Conditional options derived from another field (single option when read-only) |

The `dependsOn` object specifies conditional options for this field based on
another field's value. It has:

| Field         | Required | Type   | Description                                                              |
| :------------ | :------- | :----- | :----------------------------------------------------------------------- |
| path          | Yes      | string | JSON path of the field this one depends on (e.g., `region`)              |
| allowedValues | Yes      | object | If the field at path equals key K, this field's options are the array at |
|               |          |        | `allowedValues[K]`.                                                      |

When `dependsOn` is set, the field's options are derived from the field at path.
Each `allowedValues` entry is the list of options for that key (one or more). If
the field at path has a value with no corresponding key in `allowedValues`,
there are no value restrictions for this field. UIs use this to show the right
options; the chosen or derived values are sent when ordering the catalog item.

Object keys are always strings. When the field at path is a boolean or number,
use the JSON string representation as the key.

For example, to model `backup.retention_days` (retention in days) depending on
`backup.enabled`:

```yaml
- path: backup.retention_days
  displayName: Retention (days)
  editable: true
  dependsOn:
    path: backup.enabled
    allowedValues:
      "true": ["7", "30", "90"]
      "false": ["0"]
```

Fields not listed are neither editable nor have default values. The catalog item
owner must ensure all mandatory fields are listed.

The field `requiresResources` controls provisioning order between blueprint
resources. Do not confuse it with `dependsOn` on a field entry, which controls
conditional field options based on another field's value within the same
resource (for example `version` options when `engine` changes).

##### Field paths (catalog authoring)

Field `path` values are relative to the service type spec for that resource. The
resource's `serviceType` determines which OpenAPI schema applies.

| Mechanism                   | Convention                               | Example                                   |
| :-------------------------- | :--------------------------------------- | :---------------------------------------- |
| `resources[].fields[].path` | Relative to that resource's spec         | `engine`, `vcpu.count`, `image.reference` |
| `dependsOn.path`            | Relative within the same resource's spec | `engine`                                  |
| `requiresResources`         | Blueprint resource `name`                | `[ordersDb]`                              |

Duplicate `serviceType` values in one catalog item (for example two `database`
resources) are unambiguous. The `fields` are scoped by their parent resource
block. The `userValues` and CEL references use unique resource `name` field. If
any `name` duplication occurs, the catalog instance request will fail validation
and be rejected.

##### Example: Dev Application (multi-resource)

```yaml
apiVersion: v1alpha1
kind: CatalogItem
metadata:
  name: dev-container-db
  displayName: "Dev Application"
spec:
  resources:
    - name: ordersDb
      serviceType: database
      fields:
        - path: "engine"
          editable: true
          default: postgres
          validationSchema: { enum: [postgres, mysql] }
        - path: "version"
          editable: true
          default: "16"
          dependsOn:
            path: engine
            allowedValues:
              postgres: ["14", "15", "16", "17"]
              mysql: ["8.0"]
        - path: "resources.cpu"
          default: 1
        - path: "resources.memory"
          default: 512MB
        - path: "resources.storage"
          default: 10GB
        - path: "metadata.name"
          default: orders-db

    - name: app
      serviceType: container
      requiresResources: [ordersDb]
      fields:
        - path: "image.reference"
          editable: false
          default: registry.example.com/orders-api:1.0
        - path: "metadata.name"
          default: orders-api
        - path: "process.env[0].name"
          default: DATABASE_URL
        - path: "process.env[0].value"
          default: "${ordersDb.connectionString}"
        - path: "network.ports[0].container_port"
          default: 8080
        - path: "network.ports[0].visibility"
          default: internal
```

The `DATABASE_URL` / `${ordersDb.connectionString}` pair illustrates a fixed env
name (catalog default) and an env value wired from a dependency output via CEL
once database outputs exist.

#### CatalogItemInstance

A CatalogItemInstance is a user's order against a catalog item. It references
the catalog item by id and supplies optional user values that override editable
fields. Creating an instance triggers catalog resolution, which produces the
effective resource graph sent to placement.

##### CatalogItemInstance spec

| Field         | Required | Type   | Description                                        |
| :------------ | :------- | :----- | :------------------------------------------------- |
| catalogItemId | Yes      | string | Catalog item to provision (immutable after create) |
| userValues    | Yes      | array  | User overrides for editable fields                 |

Each `userValue`:

| Field    | Required | Description                                                         |
| :------- | :------- | :------------------------------------------------------------------ |
| resource | Yes      | Blueprint `name`; identifies which resource the override applies to |
| path     | Yes      | Relative field path (same convention as catalog `fields[].path`)    |
| value    | Yes      | Value for that field                                                |

Example for a single-resource VM (`name: main`):

```yaml
kind: CatalogItemInstance
spec:
  catalogItemId: dev-vm
  userValues:
    - resource: webserver
      path: vcpu.count
      value: 4
```

Example for a multi-resource application:

```yaml
kind: CatalogItemInstance
spec:
  catalogItemId: dev-container-db
  userValues:
    - resource: ordersDb
      path: version
      value: "17"
    - resource: app
      path: "image.reference"
      value: "registry.example.com/orders-api:1.0"
```

##### Catalog resolution

Catalog resolution turns a `CatalogItemInstance` into an effective resource
graph ready for placement. Each graph node is a provision-able resource: a
service-type shaped spec built from the `serviceType` template, catalog field
defaults, and user overrides.

###### Per-resource transformation

For each blueprint resource being resolved:

1. Select the service type: Read `serviceType` from the blueprint entry. Load
   the matching service type from the registry. This defines the OpenAPI schema
   and baseline `spec` template for that node.

2. Validate: Check that catalog `fields` paths are valid for that schema;
   defaults and `userValues` satisfy `validation_schema` and `depends_on` rules;
   each `userValue` references a known blueprint `name` and relative `path`.

3. Transform/Merge into an effective spec: — Start from a copy of the
   ServiceType template, overlay catalog `fields[].default`, then overlay
   matching `userValues` for editable paths.

4. Add the resolved resource to the graph: Combine the merged spec with the
   node's identity: blueprint `name`, `serviceType`, and `requiresResources`.

The result is the service type instance spec for each resource.

###### CEL and cross-resource wiring

| Mechanism     | Convention            | Example                        |
| :------------ | :-------------------- | :----------------------------- |
| CEL (outputs) | `${name.outputField}` | `${ordersDb.connectionString}` |

CEL references in catalog field defaults are not user input. They refer to
outputs published when the source resource reaches `Ready` state. Placement
resolves them in a second phase after dependency outputs exist (see
[Declarative API](/enhancements/declarative-api/declarative-api.md)). Placement
also infers DAG edges from CEL references alongside `requiresResources`.

Defining standard outputs on service types (for example `connectionString`,
`host`, `port`) is follow-up work.

###### Example: Placement payload (effective graph after catalog resolution)

```json
{
  "apiVersion": "v1alpha1",
  "catalogItemInstanceId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "spec": {
    "resources": [
      {
        "name": "ordersDb",
        "serviceType": "database",
        "requiresResources": [],
        "spec": {
          "serviceType": "database",
          "engine": "postgres",
          "version": "17",
          "resources": {
            "cpu": 1,
            "memory": "512MB",
            "storage": "10GB"
          },
          "metadata": {
            "name": "orders-db"
          }
        }
      },
      {
        "name": "app",
        "serviceType": "container",
        "requiresResources": ["ordersDb"],
        "spec": {
          "serviceType": "container",
          "image": {
            "reference": "registry.example.com/orders-api:1.0"
          },
          "metadata": {
            "name": "orders-api"
          },
          "process": {
            "env": [
              {
                "name": "DATABASE_URL",
                "value": "${ordersDb.connectionString}"
              }
            ]
          },
          "network": {
            "ports": [
              {
                "containerPort": 8080,
                "visibility": "internal"
              }
            ]
          }
        }
      }
    ]
  }
}
```

#### Versioning

The **`apiVersion`** field versions the CatalogItem schema itself (e.g.,
`v1alpha1`), enabling evolution of the CatalogItem structure.

## Design Details

### Validation

The _validationSchema_ field follows
[JSON Schema (draft 2020-12)](https://json-schema.org/draft/2020-12/json-schema-validation).  
This
standard supports:

- Numeric constraints: _minimum, maximum, multipleOf_
- String patterns: _pattern, minLength, maxLength_
- Enumerations: _enum_
- Array constraints: _minItems, maxItems_
- Conditional logic: _if/then/else_

For the complete validation vocabulary, see the
[JSON Schema Validation specification](https://json-schema.org/draft/2020-12/json-schema-validation).

### Data Flow

#### Catalog item (authoring)

1. Admin creates a CatalogItem: `resources[]` blueprint with per-resource
   `fields`, `requiresResources`, and service type references.
2. Catalog item validation runs at create/update (field paths, `depends_on`,
   `requiresResources`, service type references, blueprint immutability).

#### Catalog item instance (order and resolution)

1. User submits a CatalogItemInstance: `catalogItemId` and optional `userValues`
   for editable fields.
2. Catalog resolution: For each blueprint resource, load ServiceType template,
   validate, merge catalog defaults and user overrides, assemble the effective
   resource graph. Unresolved CEL remains in the spec for placement.
3. Placement: Catalog sends the full graph to placement. Placement builds the
   DAG from `requiresResources` and CEL edges, evaluate policy per node,
   provision per DAG level via SPRM.

See [Declarative API](/enhancements/declarative-api/declarative-api.md) for CEL
two-phase evaluation, DAG levels, and status-driven progression.

Note: The validationSchema is used by both UI (for UX) and DCM (for
enforcement). Users may bypass the UI (CLI, Ansible, cURL), so DCM must always
validate.
