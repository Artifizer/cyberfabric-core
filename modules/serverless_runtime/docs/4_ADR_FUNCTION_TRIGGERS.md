# ADR — Function Triggers (Schedules, Events & Trigger Mechanisms)

## Status
Proposed

## Context
The Serverless Runtime supports multiple trigger mechanisms for starting function and workflow executions. This ADR defines:
- Trigger types and their semantics
- Schedule-based triggers (cron)
- Event-driven triggers (subscription to platform events)
- Webhook triggers (external HTTP callbacks)
- Internal triggers (function-to-function calls)

This ADR complements `2_ADR_FUNCTION_REGISTRY.md` (definition management) and `3_ADR_FUNCTION_ENTRYPOINT.md` (invocation APIs).

## Requirements Addressed
| Requirement | Description |
|-------------|-------------|
| BR-007 | Trigger mechanisms (schedule, API, event, internal) |
| BR-023 | Schedule lifecycle and missed schedule handling |
| BR-110 | Schedule-level input parameters and overrides |
| BR-012 | Graceful disconnection handling |

## Trigger Types Overview

| Trigger Type | Description | Use Cases |
|--------------|-------------|-----------|
| **API** | Direct invocation via JSON-RPC or Jobs API | On-demand execution, user actions |
| **Schedule** | Time-based recurring execution (cron) | Reports, cleanup jobs, periodic sync |
| **Event** | Subscription to platform events | Reactive workflows, integrations |
| **Webhook** | External HTTP callback endpoint | Third-party integrations, webhooks |
| **Internal** | Function-to-function invocation | Composition, helper functions |

---

## Schedule-Based Triggers

Schedules define recurring execution of functions/workflows based on time.

### Schedule Model

**Schema ID (type):** `gts.x.core.faas.schedule.v1~`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "gts://gts.x.core.faas.schedule.v1~",
  "title": "FaaS Schedule",
  "type": "object",
  "properties": {
    "schedule_id": {
      "type": "string",
      "description": "Unique identifier within tenant scope"
    },
    "definition_id": {
      "type": "string",
      "description": "GTS ID of the function/workflow to execute"
    },
    "cron": {
      "type": "string",
      "description": "Cron expression (standard 5-field or 6-field with seconds)"
    },
    "timezone": {
      "type": "string",
      "default": "UTC",
      "description": "IANA timezone for schedule evaluation"
    },
    "params": {
      "type": "object",
      "description": "Default input parameters for each execution"
    },
    "execution_context": {
      "type": "string",
      "enum": ["system", "api_client", "user"],
      "default": "system",
      "description": "Security context for scheduled executions"
    },
    "enabled": {
      "type": "boolean",
      "default": true
    },
    "missed_schedule_policy": {
      "type": "string",
      "enum": ["skip", "catch_up"],
      "default": "skip",
      "description": "How to handle missed schedules during downtime"
    },
    "max_catch_up_runs": {
      "type": "integer",
      "minimum": 1,
      "maximum": 100,
      "default": 1,
      "description": "Maximum catch-up executions when policy is catch_up"
    },
    "concurrency_policy": {
      "type": "string",
      "enum": ["allow", "forbid", "replace"],
      "default": "allow",
      "description": "Behavior when previous execution is still running"
    }
  },
  "required": ["schedule_id", "definition_id", "cron"]
}
```

### Cron Expression Format

Schedules use standard cron expressions:

```
┌───────────── minute (0-59)
│ ┌───────────── hour (0-23)
│ │ ┌───────────── day of month (1-31)
│ │ │ ┌───────────── month (1-12 or JAN-DEC)
│ │ │ │ ┌───────────── day of week (0-6 or SUN-SAT)
│ │ │ │ │
* * * * *
```

**Examples:**
| Expression | Description |
|------------|-------------|
| `0 2 * * *` | Every day at 2:00 AM |
| `*/15 * * * *` | Every 15 minutes |
| `0 9 * * MON-FRI` | Weekdays at 9:00 AM |
| `0 0 1 * *` | First day of every month at midnight |
| `0 */4 * * *` | Every 4 hours |

### Concurrency Policies

| Policy | Behavior |
|--------|----------|
| `allow` | Start new execution even if previous is running |
| `forbid` | Skip this scheduled run if previous is still running |
| `replace` | Cancel previous execution and start new one |

### Missed Schedule Handling

When the scheduler is unavailable (maintenance, outage), schedules may be missed.

| Policy | Behavior |
|--------|----------|
| `skip` | Ignore missed schedules; resume from next scheduled time |
| `catch_up` | Execute missed schedules up to `max_catch_up_runs` limit |

**Example:** A daily schedule at 2 AM with `catch_up` policy and `max_catch_up_runs: 3`:
- If scheduler is down for 5 days, only 3 catch-up executions occur
- Executions run sequentially with a brief delay between each

---

## Schedule Management APIs

### Base URL
`/api/serverless-runtime/v1`

### 1) Create Schedule

`POST /schedules`

**Request:**
```json
{
  "schedule_id": "daily-tax-report",
  "definition_id": "gts.x.core.faas.func.v1~vendor.app.billing.generate_tax_report.v1~",
  "cron": "0 2 * * *",
  "timezone": "America/Los_Angeles",
  "params": {
    "report_type": "daily",
    "format": "pdf"
  },
  "execution_context": "system",
  "enabled": true,
  "missed_schedule_policy": "skip",
  "max_catch_up_runs": 3,
  "concurrency_policy": "forbid"
}
```

**Response:**
`201 Created`
```json
{
  "schedule_id": "daily-tax-report",
  "tenant_id": "tenant_abc",
  "definition_id": "gts.x.core.faas.func.v1~vendor.app.billing.generate_tax_report.v1~",
  "cron": "0 2 * * *",
  "timezone": "America/Los_Angeles",
  "enabled": true,
  "next_run_at": "2026-01-28T10:00:00.000Z",
  "created_at": "2026-01-27T10:00:00.000Z"
}
```

**Error Responses:**

`400 Bad Request` — Invalid cron expression
```json
{
  "error": "invalid_cron",
  "message": "Invalid cron expression: '0 25 * * *' - hour must be 0-23",
  "field": "cron"
}
```

`404 Not Found` — Definition not found
```json
{
  "error": "definition_not_found",
  "message": "Definition not found or not accessible",
  "definition_id": "gts.x.core.faas.func.v1~vendor.app.billing.missing.v1~"
}
```

`409 Conflict` — Schedule already exists
```json
{
  "error": "schedule_exists",
  "message": "A schedule with this ID already exists",
  "schedule_id": "daily-tax-report"
}
```

---

### 2) Get Schedule

`GET /schedules/{schedule_id}`

**Response:**
`200 OK`
```json
{
  "schedule_id": "daily-tax-report",
  "tenant_id": "tenant_abc",
  "definition_id": "gts.x.core.faas.func.v1~vendor.app.billing.generate_tax_report.v1~",
  "cron": "0 2 * * *",
  "timezone": "America/Los_Angeles",
  "params": {
    "report_type": "daily",
    "format": "pdf"
  },
  "execution_context": "system",
  "enabled": true,
  "missed_schedule_policy": "skip",
  "max_catch_up_runs": 3,
  "concurrency_policy": "forbid",
  "next_run_at": "2026-01-28T10:00:00.000Z",
  "last_run_at": "2026-01-27T10:00:00.000Z",
  "last_run_status": "succeeded",
  "created_at": "2026-01-20T10:00:00.000Z",
  "updated_at": "2026-01-27T10:00:00.000Z"
}
```

---

### 3) Update Schedule

`PATCH /schedules/{schedule_id}`

Updates schedule configuration. Changes take effect from the next scheduled run.

**Request:**
```json
{
  "cron": "0 3 * * *",
  "params": {
    "report_type": "daily",
    "format": "csv"
  }
}
```

**Response:**
`200 OK`
```json
{
  "schedule_id": "daily-tax-report",
  "updated_fields": ["cron", "params"],
  "next_run_at": "2026-01-28T11:00:00.000Z",
  "updated_at": "2026-01-27T12:00:00.000Z"
}
```

---

### 4) Pause Schedule

`POST /schedules/{schedule_id}:pause`

Pauses a schedule. No new executions will be triggered until resumed.

**Response:**
`200 OK`
```json
{
  "schedule_id": "daily-tax-report",
  "enabled": false,
  "paused_at": "2026-01-27T12:00:00.000Z",
  "paused_by": "user_123"
}
```

---

### 5) Resume Schedule

`POST /schedules/{schedule_id}:resume`

Resumes a paused schedule.

**Request (optional):**
```json
{
  "catch_up_missed": true
}
```

**Response:**
`200 OK`
```json
{
  "schedule_id": "daily-tax-report",
  "enabled": true,
  "resumed_at": "2026-01-27T14:00:00.000Z",
  "resumed_by": "user_123",
  "next_run_at": "2026-01-28T10:00:00.000Z",
  "catch_up_runs_queued": 0
}
```

---

### 6) Delete Schedule

`DELETE /schedules/{schedule_id}`

**Query Parameters:**
| Parameter | Type | Description |
|-----------|------|-------------|
| `cancel_in_flight` | boolean | Cancel any in-flight executions triggered by this schedule (default: `false`) |

**Response:**
`200 OK`
```json
{
  "schedule_id": "daily-tax-report",
  "deleted_at": "2026-01-27T12:00:00.000Z",
  "deleted_by": "user_123"
}
```

---

### 7) List Schedules

`GET /schedules`

**Query Parameters:**
| Parameter | Type | Description |
|-----------|------|-------------|
| `definition_id` | string | Filter by definition |
| `enabled` | boolean | Filter by enabled state |
| `search` | string | Search in schedule_id |
| `limit` | integer | Maximum results (default: 25, max: 100) |
| `cursor` | string | Pagination cursor |

**Response:**
`200 OK`
```json
{
  "items": [
    {
      "schedule_id": "daily-tax-report",
      "definition_id": "gts.x.core.faas.func.v1~vendor.app.billing.generate_tax_report.v1~",
      "cron": "0 2 * * *",
      "timezone": "America/Los_Angeles",
      "enabled": true,
      "next_run_at": "2026-01-28T10:00:00.000Z",
      "last_run_status": "succeeded"
    }
  ],
  "page_info": {
    "next_cursor": "<opaque>",
    "prev_cursor": null,
    "limit": 25,
    "total_count": 5
  }
}
```

---

### 8) Get Schedule Run History

`GET /schedules/{schedule_id}/runs`

**Query Parameters:**
| Parameter | Type | Description |
|-----------|------|-------------|
| `status` | string | Filter by status: `succeeded`, `failed`, `skipped`, `running` |
| `since` | string | ISO 8601 timestamp - return runs after this time |
| `until` | string | ISO 8601 timestamp - return runs before this time |
| `limit` | integer | Maximum results (default: 25, max: 100) |
| `cursor` | string | Pagination cursor |

**Response:**
`200 OK`
```json
{
  "schedule_id": "daily-tax-report",
  "runs": [
    {
      "run_id": "run_001",
      "scheduled_at": "2026-01-27T10:00:00.000Z",
      "status": "succeeded",
      "job_id": "job_abc123",
      "started_at": "2026-01-27T10:00:01.000Z",
      "completed_at": "2026-01-27T10:00:15.000Z",
      "duration_ms": 14000
    },
    {
      "run_id": "run_002",
      "scheduled_at": "2026-01-26T10:00:00.000Z",
      "status": "skipped",
      "skip_reason": "concurrency_policy_forbid",
      "previous_job_id": "job_xyz789"
    },
    {
      "run_id": "run_003",
      "scheduled_at": "2026-01-25T10:00:00.000Z",
      "status": "failed",
      "job_id": "job_def456",
      "started_at": "2026-01-25T10:00:01.000Z",
      "completed_at": "2026-01-25T10:00:05.000Z",
      "error": {
        "id": "gts.x.core.faas.err.v1~x.core.errors.runtime_timeout.v1~",
        "message": "Execution timed out"
      }
    }
  ],
  "next_runs": [
    {"scheduled_at": "2026-01-28T10:00:00.000Z"},
    {"scheduled_at": "2026-01-29T10:00:00.000Z"},
    {"scheduled_at": "2026-01-30T10:00:00.000Z"}
  ],
  "page_info": {
    "next_cursor": "<opaque>",
    "prev_cursor": null,
    "limit": 25
  }
}
```

---

### 9) Trigger Schedule Now

`POST /schedules/{schedule_id}:trigger`

Manually triggers an immediate execution of the scheduled function, independent of the cron schedule.

**Request (optional):**
```json
{
  "params_override": {
    "format": "xlsx"
  }
}
```

**Response:**
`202 Accepted`
```json
{
  "schedule_id": "daily-tax-report",
  "job_id": "job_manual_001",
  "triggered_at": "2026-01-27T15:30:00.000Z",
  "triggered_by": "user_123",
  "params": {
    "report_type": "daily",
    "format": "xlsx"
  }
}
```

---

## Event-Driven Triggers

Event triggers subscribe to platform events and start function/workflow executions when matching events occur.

### Event Trigger Model

**Schema ID (type):** `gts.x.core.faas.event_trigger.v1~`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "gts://gts.x.core.faas.event_trigger.v1~",
  "title": "FaaS Event Trigger",
  "type": "object",
  "properties": {
    "trigger_id": {
      "type": "string",
      "description": "Unique identifier within tenant scope"
    },
    "definition_id": {
      "type": "string",
      "description": "GTS ID of the function/workflow to execute"
    },
    "event_type_id": {
      "type": "string",
      "description": "GTS ID of the event type to subscribe to (must end with ~)"
    },
    "filter": {
      "type": "string",
      "description": "Event filter query (CEL or platform-specific syntax)"
    },
    "batch": {
      "type": "object",
      "properties": {
        "enabled": {"type": "boolean", "default": false},
        "max_size": {"type": "integer", "minimum": 1, "maximum": 1000, "default": 100},
        "max_wait_ms": {"type": "integer", "minimum": 100, "maximum": 60000, "default": 5000}
      }
    },
    "execution_context": {
      "type": "string",
      "enum": ["system", "event_source"],
      "default": "system"
    },
    "enabled": {
      "type": "boolean",
      "default": true
    },
    "retry_policy": {
      "type": "object",
      "properties": {
        "max_attempts": {"type": "integer", "minimum": 1, "maximum": 10, "default": 3},
        "backoff_ms": {"type": "integer", "minimum": 100, "default": 1000}
      }
    }
  },
  "required": ["trigger_id", "definition_id", "event_type_id"]
}
```

### Event Filter Syntax

Filters use a subset of CEL (Common Expression Language):

```
# Match orders over $100
payload.total > 100

# Match specific customer
payload.customer_id == "cust_123"

# Match orders from specific region
payload.shipping.region in ["US-CA", "US-NY"]

# Compound conditions
payload.total > 100 && payload.priority == "high"
```

### Event Batching

For high-volume events, batching groups multiple events into a single function invocation:

```json
{
  "batch": {
    "enabled": true,
    "max_size": 100,
    "max_wait_ms": 5000
  }
}
```

When batching is enabled, the function receives an array of events in `input.events` instead of a single event.

---

## Event Trigger Management APIs

### 1) Create Event Trigger

`POST /event-triggers`

**Request:**
```json
{
  "trigger_id": "order-placed-handler",
  "definition_id": "gts.x.core.faas.func.v1~vendor.app.orders.process_order.v1~",
  "event_type_id": "gts.x.core.events.event.v1~vendor.app.commerce.order_placed.v1~",
  "filter": "payload.total > 100",
  "execution_context": "system",
  "enabled": true,
  "retry_policy": {
    "max_attempts": 3,
    "backoff_ms": 1000
  }
}
```

**Response:**
`201 Created`
```json
{
  "trigger_id": "order-placed-handler",
  "tenant_id": "tenant_abc",
  "definition_id": "gts.x.core.faas.func.v1~vendor.app.orders.process_order.v1~",
  "event_type_id": "gts.x.core.events.event.v1~vendor.app.commerce.order_placed.v1~",
  "filter": "payload.total > 100",
  "enabled": true,
  "subscription_status": "active",
  "created_at": "2026-01-27T10:00:00.000Z"
}
```

---

### 2) Get Event Trigger

`GET /event-triggers/{trigger_id}`

---

### 3) Update Event Trigger

`PATCH /event-triggers/{trigger_id}`

---

### 4) Enable/Disable Event Trigger

`POST /event-triggers/{trigger_id}:enable`
`POST /event-triggers/{trigger_id}:disable`

---

### 5) Delete Event Trigger

`DELETE /event-triggers/{trigger_id}`

---

### 6) List Event Triggers

`GET /event-triggers`

**Query Parameters:**
| Parameter | Type | Description |
|-----------|------|-------------|
| `definition_id` | string | Filter by definition |
| `event_type_id` | string | Filter by event type |
| `enabled` | boolean | Filter by enabled state |
| `limit` | integer | Maximum results (default: 25, max: 100) |
| `cursor` | string | Pagination cursor |

---

### 7) Get Event Trigger Metrics

`GET /event-triggers/{trigger_id}/metrics`

**Response:**
`200 OK`
```json
{
  "trigger_id": "order-placed-handler",
  "period": "24h",
  "metrics": {
    "events_received": 1542,
    "events_filtered_out": 342,
    "executions_triggered": 1200,
    "executions_succeeded": 1180,
    "executions_failed": 20,
    "avg_latency_ms": 45,
    "p99_latency_ms": 120
  }
}
```

---

## Webhook Triggers

Webhook triggers expose HTTP endpoints that external systems can call to trigger function executions.

### Webhook Trigger Model

**Schema ID (type):** `gts.x.core.faas.webhook_trigger.v1~`

### 1) Create Webhook Trigger

`POST /webhook-triggers`

**Request:**
```json
{
  "trigger_id": "github-push-handler",
  "definition_id": "gts.x.core.faas.func.v1~vendor.app.ci.handle_push.v1~",
  "authentication": {
    "type": "hmac_sha256",
    "secret_ref": "github_webhook_secret"
  },
  "allowed_sources": ["192.30.252.0/22", "185.199.108.0/22"],
  "enabled": true
}
```

**Response:**
`201 Created`
```json
{
  "trigger_id": "github-push-handler",
  "tenant_id": "tenant_abc",
  "webhook_url": "https://api.example.com/webhooks/tenant_abc/github-push-handler",
  "authentication": {
    "type": "hmac_sha256",
    "header": "X-Hub-Signature-256"
  },
  "enabled": true,
  "created_at": "2026-01-27T10:00:00.000Z"
}
```

### Webhook Authentication Types

| Type | Description |
|------|-------------|
| `none` | No authentication (not recommended) |
| `hmac_sha256` | HMAC-SHA256 signature verification |
| `hmac_sha1` | HMAC-SHA1 signature verification (legacy) |
| `basic` | HTTP Basic authentication |
| `bearer` | Bearer token authentication |
| `api_key` | API key in header or query parameter |

---

### 2) Get Webhook Trigger

`GET /webhook-triggers/{trigger_id}`

---

### 3) Regenerate Webhook Secret

`POST /webhook-triggers/{trigger_id}:regenerate-secret`

---

### 4) List Webhook Triggers

`GET /webhook-triggers`

---

### 5) Delete Webhook Trigger

`DELETE /webhook-triggers/{trigger_id}`

---

## Internal Triggers (Function-to-Function)

Internal triggers allow functions to invoke other functions as part of their execution.

### Direct Invocation

From within a Starlark function, use `r_invoke_v1()`:

```python
def main(ctx, input):
    # Invoke another function synchronously
    tax_result = r_await(r_invoke_v1(
        "gts.x.core.faas.func.v1~vendor.app.billing.calculate_tax.v1~",
        params = {"invoice_total": input.subtotal, "region": input.region},
        timeout_ms = 5000,
    ))

    return {
        "subtotal": input.subtotal,
        "tax": tax_result.value.tax,
        "total": input.subtotal + tax_result.value.tax
    }
```

### Entrypoint Trait

Functions can be marked as internal-only using the `entrypoint` trait:

```json
{
  "traits": {
    "entrypoint": false
  }
}
```

When `entrypoint: false`:
- Function cannot be invoked via external APIs (JSON-RPC, Jobs API)
- Function can only be invoked internally via `r_invoke_v1()`
- Useful for helper functions and shared logic

---

## Disconnection Handling (BR-012)

When an external dependency (event source, webhook provider, etc.) is disconnected:

### Behavior

1. **New executions rejected:** Trigger returns an error for new invocations
2. **In-flight executions:** Allowed to complete or fail gracefully
3. **Reconnection:** Automatic retry with exponential backoff

### Trigger Status

Triggers report connection status:

```json
{
  "trigger_id": "order-placed-handler",
  "enabled": true,
  "connection_status": "degraded",
  "connection_details": {
    "event_broker": "disconnected",
    "last_connected_at": "2026-01-27T09:55:00.000Z",
    "reconnect_attempts": 3,
    "next_reconnect_at": "2026-01-27T10:05:00.000Z"
  }
}
```

### Status Values

| Status | Description |
|--------|-------------|
| `active` | Fully operational |
| `degraded` | Partial connectivity issues |
| `disconnected` | Cannot receive events/webhooks |
| `paused` | Manually paused by user |

---

## Audit Events

All trigger operations emit audit events:

| Event Type | Description |
|------------|-------------|
| `gts.x.core.faas.audit.v1~x.core.triggers.schedule_created.v1~` | Schedule created |
| `gts.x.core.faas.audit.v1~x.core.triggers.schedule_updated.v1~` | Schedule updated |
| `gts.x.core.faas.audit.v1~x.core.triggers.schedule_paused.v1~` | Schedule paused |
| `gts.x.core.faas.audit.v1~x.core.triggers.schedule_resumed.v1~` | Schedule resumed |
| `gts.x.core.faas.audit.v1~x.core.triggers.schedule_deleted.v1~` | Schedule deleted |
| `gts.x.core.faas.audit.v1~x.core.triggers.schedule_triggered.v1~` | Schedule manually triggered |
| `gts.x.core.faas.audit.v1~x.core.triggers.event_trigger_created.v1~` | Event trigger created |
| `gts.x.core.faas.audit.v1~x.core.triggers.event_trigger_updated.v1~` | Event trigger updated |
| `gts.x.core.faas.audit.v1~x.core.triggers.event_trigger_deleted.v1~` | Event trigger deleted |
| `gts.x.core.faas.audit.v1~x.core.triggers.webhook_created.v1~` | Webhook trigger created |
| `gts.x.core.faas.audit.v1~x.core.triggers.webhook_deleted.v1~` | Webhook trigger deleted |
| `gts.x.core.faas.audit.v1~x.core.triggers.webhook_secret_regenerated.v1~` | Webhook secret regenerated |

---

## Security Considerations

### Schedule Security

1. **Execution context:** Scheduled executions run with configured identity (system, user, or API client)
2. **Least privilege:** System context should have minimal required permissions
3. **Audit trail:** All scheduled executions are logged with trigger correlation

### Event Trigger Security

1. **Tenant isolation:** Event subscriptions are scoped to tenant's accessible events
2. **Filter validation:** CEL filters are validated and sandboxed
3. **Rate limiting:** Per-trigger rate limits prevent event storms

### Webhook Security

1. **Authentication required:** Production webhooks should use signature verification
2. **IP allowlisting:** Optional source IP restrictions
3. **Secret rotation:** Support for secret regeneration without downtime
4. **Payload validation:** Incoming payloads validated against expected schema

---

## Alternatives Considered

### Alternative 1: Unified Trigger Model
Single trigger type that handles all mechanisms.

**Rejected because:** Different trigger types have fundamentally different configurations and semantics.

### Alternative 2: External Scheduler Service
Use a separate scheduling service (e.g., Kubernetes CronJobs).

**Rejected because:** Tighter integration needed for tenant isolation, observability, and missed schedule handling.

### Alternative 3: Polling-Based Events
Functions poll for events instead of push-based subscriptions.

**Rejected because:** Higher latency, more resource usage, doesn't scale well.

---

## Future Considerations

- **Complex schedules:** Support for more expressive schedule definitions (e.g., "second Tuesday of each month")
- **Event replay:** Ability to replay historical events through triggers for testing/recovery
- **Trigger chaining:** Define trigger chains where one function's output triggers another
- **Dead letter triggers:** Automatic triggers for DLQ events
