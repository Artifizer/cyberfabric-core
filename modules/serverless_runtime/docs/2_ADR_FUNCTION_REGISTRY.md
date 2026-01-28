# ADR — Function Registry (Definition Management & Tenant Isolation)

## Status
Proposed

## Context
The Serverless Runtime module requires a registry system to manage function and workflow definitions across tenants and users. This ADR defines:
- Registry architecture and tenant isolation model
- Definition lifecycle APIs (registration, update, versioning, deletion)
- Tenant enablement and quota management

This ADR complements `3_ADR_FUNCTION_ENTRYPOINT.md` (invocation APIs), `4_ADR_FUNCTION_TRIGGERS.md` (schedules and event triggers), and `5_ADR_STARLARK_RUNTIME.md` (execution semantics).

## Requirements Addressed
| Requirement | Description |
|-------------|-------------|
| BR-001 | Runtime authoring without platform rebuild |
| BR-002 | Tenant and user registries |
| BR-011 | Function/Workflow definition validation |
| BR-019 | Workflow/function definition versioning |
| BR-021 | Tenant enablement and isolation provisioning |
| BR-030 | Workflow/function execution isolation during updates |

See `4_ADR_FUNCTION_TRIGGERS.md` for BR-007 (trigger mechanisms) and BR-023 (schedule lifecycle).

## Registry Architecture

### Hierarchical Namespace Model

Function and workflow definitions are organized in a hierarchical namespace model that enforces tenant isolation:

```
Platform (vendor: x)
└── Application (package)
    └── Tenant (tenant_id)
        └── User (user_id, optional)
            └── Definition (namespace.type.version)
```

### GTS Identifier Structure for Registry

Definitions are identified using GTS identifiers with tenant context encoded in the namespace:

**Platform-provided definitions (shared across all tenants):**
```
gts.x.core.faas.func.v1~x.<package>.<namespace>.<type>.v<N>~
```

**Tenant-scoped definitions:**
```
gts.x.core.faas.func.v1~x.<package>.tenant.<tenant_id>.<namespace>.<type>.v<N>~
```

**User-scoped definitions (within a tenant):**
```
gts.x.core.faas.func.v1~x.<package>.tenant.<tenant_id>.user.<user_id>.<namespace>.<type>.v<N>~
```

### Visibility and Access Rules

| Definition Scope | Visible To | Modifiable By |
|------------------|------------|---------------|
| Platform | All tenants | Platform operators only |
| Tenant | Users within tenant | Tenant administrators |
| User | Owner user only | Owner user |

Tenant-scoped definitions MAY be marked as `shared: true` to allow other users within the same tenant to invoke them (but not modify them).

## Definition Lifecycle

### States

```
                    ┌─────────────┐
                    │   DRAFT     │
                    └──────┬──────┘
                           │ validate & publish
                           ▼
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  DISABLED   │◄────│   ACTIVE    │────►│  DEPRECATED │
└─────────────┘     └─────────────┘     └─────────────┘
      │                    │                   │
      │                    │                   │
      └────────────────────┴───────────────────┘
                           │
                           ▼
                    ┌─────────────┐
                    │   DELETED   │
                    └─────────────┘
```

| State | Description | Invocable | Modifiable |
|-------|-------------|-----------|------------|
| `DRAFT` | Work in progress, not yet published | No | Yes |
| `ACTIVE` | Published and available for invocation | Yes | No (create new version) |
| `DISABLED` | Temporarily unavailable | No | Yes (to re-enable) |
| `DEPRECATED` | Marked for removal, existing invocations allowed | Yes (with warning) | No |
| `DELETED` | Soft-deleted, retained for audit | No | No |

### Versioning Model

Definitions follow semantic versioning aligned with GTS conventions:

- **Major version** (v1, v2, ...): Breaking changes to `params`, `returns`, or `errors` schemas
- **Minor version** (v1.1, v1.2, ...): Backward-compatible changes (implementation updates, new optional fields)

**Version Resolution:**
- Invocations MUST specify at least the major version
- If only major version is specified, the runtime resolves to the latest active minor version
- In-flight executions are pinned to the exact version at start time (BR-030)

**Example:**
```
gts.x.core.faas.func.v1~vendor.app.billing.calculate_tax.v2~      # Latest v2.x
gts.x.core.faas.func.v1~vendor.app.billing.calculate_tax.v2.3~    # Exact v2.3
```

## Management APIs

### Base URL
`/api/serverless-runtime/v1`

### Authentication & Authorization
All management APIs require authentication. Authorization is enforced based on:
- Tenant context (from authentication token)
- User context (from authentication token)
- Required permissions per operation

---

### 1) Register Definition (Create Draft)

`POST /definitions`

Creates a new function or workflow definition in `DRAFT` state.

**Request:**
```json
{
  "definition": {
    "$schema": "https://json-schema.org/draft/2020-12/schema",
    "$id": "gts://gts.x.core.faas.func.v1~vendor.app.billing.calculate_tax.v1~",
    "allOf": [
      {"$ref": "gts://gts.x.core.faas.func.v1~"}
    ],
    "type": "object",
    "properties": {
      "id": {"const": "gts.x.core.faas.func.v1~vendor.app.billing.calculate_tax.v1~"},
      "params": {
        "const": {
          "type": "object",
          "properties": {
            "invoice_total": {"type": "number"},
            "region": {"type": "string"}
          },
          "required": ["invoice_total", "region"]
        }
      },
      "returns": {
        "const": {
          "type": "object",
          "properties": {
            "tax": {"type": "number"}
          },
          "required": ["tax"]
        }
      },
      "traits": {
        "type": "object",
        "properties": {
          "runtime": {"const": "starlark"},
          "invocation": {
            "type": "object",
            "properties": {
              "supported": {"const": ["sync", "async"]},
              "default": {"const": "sync"}
            }
          }
        }
      },
      "implementation": {
        "type": "object",
        "properties": {
          "code": {
            "type": "object",
            "properties": {
              "language": {"const": "starlark"},
              "source": {"const": "def main(ctx, input):\n  return {\"tax\": input.invoice_total * 0.0725}\n"}
            }
          }
        }
      }
    }
  },
  "scope": "tenant",
  "shared": false
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `definition` | object | Yes | The GTS-compliant function/workflow definition |
| `scope` | string | No | `"platform"`, `"tenant"`, or `"user"` (default: `"user"`) |
| `shared` | boolean | No | If `true`, other users in the tenant can invoke this definition (default: `false`) |

**Response:**
`201 Created`
```json
{
  "definition_id": "gts.x.core.faas.func.v1~vendor.app.billing.calculate_tax.v1~",
  "state": "DRAFT",
  "version": "v1",
  "scope": "tenant",
  "tenant_id": "tenant_abc",
  "owner_id": "user_123",
  "shared": false,
  "created_at": "2026-01-27T10:00:00.000Z",
  "updated_at": "2026-01-27T10:00:00.000Z",
  "validation": {
    "status": "pending"
  }
}
```

**Error Responses:**

`400 Bad Request` — Invalid definition schema
```json
{
  "error": "validation_failed",
  "message": "Definition schema validation failed",
  "details": {
    "errors": [
      {"path": "$.properties.params", "message": "params schema is required"},
      {"path": "$.properties.traits.properties.runtime", "message": "runtime is required"}
    ]
  }
}
```

`409 Conflict` — Definition already exists
```json
{
  "error": "definition_exists",
  "message": "A definition with this ID already exists",
  "definition_id": "gts.x.core.faas.func.v1~vendor.app.billing.calculate_tax.v1~"
}
```

---

### 2) Validate Definition

`POST /definitions/{definition_id}:validate`

Validates a draft definition without publishing. Performs:
- JSON Schema validation
- Code syntax validation (parse AST)
- Runtime helper usage validation
- Policy checks (allowed built-ins, resource limits)
- System-aware validation (URL allowlists, event type registration, etc.)

**Response:**
`200 OK`
```json
{
  "definition_id": "gts.x.core.faas.func.v1~vendor.app.billing.calculate_tax.v1~",
  "valid": true,
  "warnings": [],
  "info": {
    "detected_features": ["sync_invocation", "http_client"],
    "estimated_memory_mb": 64,
    "uses_workflow_primitives": false
  }
}
```

`200 OK` — Validation failed (not an HTTP error)
```json
{
  "definition_id": "gts.x.core.faas.func.v1~vendor.app.billing.calculate_tax.v1~",
  "valid": false,
  "errors": [
    {
      "code": "SYNTAX_ERROR",
      "message": "Starlark syntax error: unexpected token",
      "location": {"line": 5, "column": 12},
      "source_snippet": "  return {\"tax\": input.invoice_total * }"
    },
    {
      "code": "FORBIDDEN_BUILTIN",
      "message": "Built-in 'exec' is not allowed",
      "location": {"line": 3, "column": 3}
    }
  ],
  "warnings": [
    {
      "code": "MISSING_TIMEOUT",
      "message": "HTTP call at line 7 has no explicit timeout; default (30s) will be used"
    }
  ]
}
```

---

### 3) Publish Definition

`POST /definitions/{definition_id}:publish`

Validates and transitions a draft definition to `ACTIVE` state. The definition becomes invocable.

**Request (optional):**
```json
{
  "release_notes": "Initial release of tax calculation function"
}
```

**Response:**
`200 OK`
```json
{
  "definition_id": "gts.x.core.faas.func.v1~vendor.app.billing.calculate_tax.v1~",
  "state": "ACTIVE",
  "version": "v1",
  "published_at": "2026-01-27T10:05:00.000Z",
  "release_notes": "Initial release of tax calculation function"
}
```

`400 Bad Request` — Definition not valid
```json
{
  "error": "validation_failed",
  "message": "Cannot publish: definition validation failed",
  "validation_errors": [...]
}
```

`409 Conflict` — Definition not in DRAFT state
```json
{
  "error": "invalid_state_transition",
  "message": "Cannot publish: definition is not in DRAFT state",
  "current_state": "ACTIVE"
}
```

---

### 4) Update Definition (Create New Version)

`POST /definitions/{definition_id}:update`

Creates a new version of an existing definition. The new version starts in `DRAFT` state.

**Request:**
```json
{
  "definition": {
    "...updated definition..."
  },
  "version_bump": "minor"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `definition` | object | Yes | The updated definition |
| `version_bump` | string | No | `"major"` or `"minor"` (default: auto-detect based on schema changes) |

**Response:**
`201 Created`
```json
{
  "definition_id": "gts.x.core.faas.func.v1~vendor.app.billing.calculate_tax.v1.1~",
  "previous_version": "gts.x.core.faas.func.v1~vendor.app.billing.calculate_tax.v1~",
  "state": "DRAFT",
  "version": "v1.1",
  "version_bump": "minor",
  "created_at": "2026-01-27T11:00:00.000Z"
}
```

---

### 5) Get Definition

`GET /definitions/{definition_id}`

Retrieves a definition by ID.

**Query Parameters:**
| Parameter | Type | Description |
|-----------|------|-------------|
| `include_source` | boolean | Include implementation source code (default: `false`) |
| `version` | string | Specific version to retrieve (default: latest) |

**Response:**
`200 OK`
```json
{
  "definition_id": "gts.x.core.faas.func.v1~vendor.app.billing.calculate_tax.v1~",
  "state": "ACTIVE",
  "version": "v1",
  "scope": "tenant",
  "tenant_id": "tenant_abc",
  "owner_id": "user_123",
  "shared": false,
  "created_at": "2026-01-27T10:00:00.000Z",
  "updated_at": "2026-01-27T10:05:00.000Z",
  "published_at": "2026-01-27T10:05:00.000Z",
  "definition": {
    "$id": "gts://gts.x.core.faas.func.v1~vendor.app.billing.calculate_tax.v1~",
    "...schema without implementation.code.source..."
  },
  "traits_summary": {
    "runtime": "starlark",
    "invocation_modes": ["sync", "async"],
    "default_invocation": "sync",
    "is_idempotent": true,
    "caching_max_age_seconds": 60
  }
}
```

---

### 6) List Definitions

`GET /definitions`

Lists definitions accessible to the authenticated user.

**Query Parameters:**
| Parameter | Type | Description |
|-----------|------|-------------|
| `scope` | string | Filter by scope: `"platform"`, `"tenant"`, `"user"`, `"all"` (default: `"all"`) |
| `state` | string | Filter by state (comma-separated): `"DRAFT"`, `"ACTIVE"`, `"DISABLED"`, `"DEPRECATED"` |
| `runtime` | string | Filter by runtime: `"starlark"`, `"wasm"`, etc. |
| `tag` | string | Filter by tag (repeatable) |
| `search` | string | Search in title, description, and ID |
| `limit` | integer | Maximum results (default: 25, max: 100) |
| `cursor` | string | Pagination cursor |

**Response:**
`200 OK`
```json
{
  "items": [
    {
      "definition_id": "gts.x.core.faas.func.v1~vendor.app.billing.calculate_tax.v1~",
      "title": "Calculate Tax",
      "description": "Calculates sales tax based on region",
      "state": "ACTIVE",
      "version": "v1",
      "scope": "tenant",
      "runtime": "starlark",
      "invocation_modes": ["sync", "async"],
      "tags": ["billing", "tax"],
      "owner_id": "user_123",
      "published_at": "2026-01-27T10:05:00.000Z"
    }
  ],
  "page_info": {
    "next_cursor": "<opaque>",
    "prev_cursor": null,
    "limit": 25,
    "total_count": 42
  }
}
```

---

### 7) List Definition Versions

`GET /definitions/{definition_id}/versions`

Lists all versions of a definition.

**Response:**
`200 OK`
```json
{
  "definition_id": "gts.x.core.faas.func.v1~vendor.app.billing.calculate_tax",
  "versions": [
    {
      "version": "v1",
      "full_id": "gts.x.core.faas.func.v1~vendor.app.billing.calculate_tax.v1~",
      "state": "DEPRECATED",
      "published_at": "2026-01-01T10:00:00.000Z",
      "deprecated_at": "2026-01-15T10:00:00.000Z"
    },
    {
      "version": "v2",
      "full_id": "gts.x.core.faas.func.v1~vendor.app.billing.calculate_tax.v2~",
      "state": "ACTIVE",
      "published_at": "2026-01-15T10:00:00.000Z"
    },
    {
      "version": "v2.1",
      "full_id": "gts.x.core.faas.func.v1~vendor.app.billing.calculate_tax.v2.1~",
      "state": "DRAFT",
      "created_at": "2026-01-27T10:00:00.000Z"
    }
  ]
}
```

---

### 8) Disable/Enable Definition

`POST /definitions/{definition_id}:disable`
`POST /definitions/{definition_id}:enable`

Toggles the `DISABLED` state.

**Response:**
`200 OK`
```json
{
  "definition_id": "gts.x.core.faas.func.v1~vendor.app.billing.calculate_tax.v1~",
  "state": "DISABLED",
  "disabled_at": "2026-01-27T12:00:00.000Z",
  "disabled_by": "user_123"
}
```

---

### 9) Deprecate Definition

`POST /definitions/{definition_id}:deprecate`

Marks a definition as deprecated. Existing invocations continue to work but emit warnings.

**Request:**
```json
{
  "reason": "Replaced by v2 with improved tax calculation",
  "successor_id": "gts.x.core.faas.func.v1~vendor.app.billing.calculate_tax.v2~",
  "sunset_date": "2026-03-01T00:00:00.000Z"
}
```

**Response:**
`200 OK`
```json
{
  "definition_id": "gts.x.core.faas.func.v1~vendor.app.billing.calculate_tax.v1~",
  "state": "DEPRECATED",
  "deprecated_at": "2026-01-27T12:00:00.000Z",
  "deprecation": {
    "reason": "Replaced by v2 with improved tax calculation",
    "successor_id": "gts.x.core.faas.func.v1~vendor.app.billing.calculate_tax.v2~",
    "sunset_date": "2026-03-01T00:00:00.000Z"
  }
}
```

---

### 10) Delete Definition

`DELETE /definitions/{definition_id}`

Soft-deletes a definition. Retained for audit purposes.

**Query Parameters:**
| Parameter | Type | Description |
|-----------|------|-------------|
| `force` | boolean | Delete even if there are in-flight executions (default: `false`) |

**Response:**
`200 OK`
```json
{
  "definition_id": "gts.x.core.faas.func.v1~vendor.app.billing.calculate_tax.v1~",
  "state": "DELETED",
  "deleted_at": "2026-01-27T12:00:00.000Z",
  "deleted_by": "user_123",
  "retention_until": "2026-04-27T12:00:00.000Z"
}
```

`409 Conflict` — In-flight executions exist
```json
{
  "error": "in_flight_executions",
  "message": "Cannot delete: 3 executions are currently in progress",
  "execution_count": 3,
  "hint": "Use ?force=true to delete anyway, or wait for executions to complete"
}
```

---

## Tenant Enablement

### Enable Serverless Runtime for Tenant

`POST /tenants/{tenant_id}/serverless-runtime:enable`

Provisions the Serverless Runtime capability for a tenant with initial quotas and governance settings.

**Request:**
```json
{
  "quotas": {
    "max_definitions": 100,
    "max_concurrent_executions": 50,
    "max_execution_duration_seconds": 86400,
    "max_memory_mb_per_execution": 512,
    "max_cpu_per_execution": 1.0,
    "execution_history_retention_days": 30
  },
  "governance": {
    "allowed_runtimes": ["starlark"],
    "require_approval_for_publish": false,
    "allowed_outbound_domains": ["*.example.com", "api.stripe.com"]
  }
}
```

**Response:**
`200 OK`
```json
{
  "tenant_id": "tenant_abc",
  "enabled": true,
  "enabled_at": "2026-01-27T10:00:00.000Z",
  "quotas": {...},
  "governance": {...}
}
```

### Get Tenant Configuration

`GET /tenants/{tenant_id}/serverless-runtime`

**Response:**
`200 OK`
```json
{
  "tenant_id": "tenant_abc",
  "enabled": true,
  "enabled_at": "2026-01-27T10:00:00.000Z",
  "quotas": {
    "max_definitions": 100,
    "max_concurrent_executions": 50,
    "...": "..."
  },
  "usage": {
    "current_definitions": 12,
    "current_concurrent_executions": 3,
    "executions_today": 1542
  },
  "governance": {...}
}
```

### Update Tenant Quotas

`PATCH /tenants/{tenant_id}/serverless-runtime`

**Request:**
```json
{
  "quotas": {
    "max_concurrent_executions": 100
  }
}
```

### Disable Serverless Runtime for Tenant

`POST /tenants/{tenant_id}/serverless-runtime:disable`

Disables the capability. Existing definitions are retained but cannot be invoked.

---

## Audit Events

All registry operations emit audit events:

| Event Type | Description |
|------------|-------------|
| `gts.x.core.faas.audit.v1~x.core.registry.definition_created.v1~` | Definition created |
| `gts.x.core.faas.audit.v1~x.core.registry.definition_published.v1~` | Definition published |
| `gts.x.core.faas.audit.v1~x.core.registry.definition_updated.v1~` | Definition updated |
| `gts.x.core.faas.audit.v1~x.core.registry.definition_disabled.v1~` | Definition disabled |
| `gts.x.core.faas.audit.v1~x.core.registry.definition_enabled.v1~` | Definition enabled |
| `gts.x.core.faas.audit.v1~x.core.registry.definition_deprecated.v1~` | Definition deprecated |
| `gts.x.core.faas.audit.v1~x.core.registry.definition_deleted.v1~` | Definition deleted |
| `gts.x.core.faas.audit.v1~x.core.registry.tenant_enabled.v1~` | Tenant enabled for serverless runtime |
| `gts.x.core.faas.audit.v1~x.core.registry.tenant_disabled.v1~` | Tenant disabled |

See `4_ADR_FUNCTION_TRIGGERS.md` for trigger-related audit events (schedules, event triggers, webhooks).

Audit events include:
- `tenant_id`
- `actor_id` (user or system identity)
- `correlation_id`
- `timestamp`
- `details` (operation-specific payload)

---

## Security Considerations

### Tenant Isolation

1. **Registry isolation:** Each tenant has a logically separate namespace. Queries are automatically filtered by tenant context.

2. **Execution isolation:** Functions from different tenants execute in isolated runtime environments with no shared state.

3. **Secret isolation:** Secrets are scoped to tenant and never exposed to other tenants' code.

### Definition Validation Security

1. **Code scanning:** Starlark code is statically analyzed for forbidden constructs before registration.

2. **URL allowlisting:** Outbound HTTP calls are validated against tenant-configured domain allowlists.

3. **Resource limits:** Definitions cannot request resources exceeding tenant quotas.

---

## Alternatives Considered

### Alternative 1: Flat Namespace with Tenant Prefixes
Instead of hierarchical namespaces, use flat IDs with tenant prefixes embedded.

**Rejected because:** Makes queries and access control more complex; harder to reason about ownership.

### Alternative 2: Separate Registries per Tenant
Deploy separate registry instances per tenant.

**Rejected because:** Operational overhead; harder to support platform-level shared definitions.

### Alternative 3: Git-based Version Control
Store definitions in a Git repository with Git-based versioning.

**Rejected because:** Adds external dependency; complicates runtime updates; not aligned with GTS-first approach.

---

## Future Considerations

- **Definition marketplace:** Allow tenants to publish definitions for discovery by other tenants
- **Definition signing:** Cryptographic signing of definitions for integrity verification
- **Blue-green deployments:** Traffic splitting between versions for gradual rollout
- **Definition templates:** Pre-built templates for common patterns
