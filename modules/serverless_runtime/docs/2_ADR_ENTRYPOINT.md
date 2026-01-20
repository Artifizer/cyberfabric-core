# ADR — Serverless Runtime Entrypoints (Function & Workflow Definitions, Invocation API)

## Status
Proposed

## Context
The Serverless Runtime module needs a stable, tenant-safe way to:
- register functions & workflows schema and code definitions at runtime
- invoke those definitions and native functions via API, regardless of implementation language
- observe execution progress and outcomes
- retrieve results

This ADR defines:
- GTS-based identifiers and JSON Schema conventions for functions & workflows definitions
- definition “traits” (semantics, limits, runtime, error types)
- invocation API (start, status, cancel, dry-run) with observability fields

## Classifications

### Executor

An executor is a native module that executes functions and/or workflows. The pluggable executor architecture allows any native module to adapt the generic function and workflow mechanisms for a specific language or declaration format. For instance, it would be possible to have a Starlark executor and a Serverless Workflow executor which each enable a different definition language or format.

Executors must support basic function execution, and may support additional workflow features as relevant.

### Function

Functions are single-unit compute entrypoints, generally designed for request/response (synchronous) execution:
- Stateless with respect to the runtime (any durable state is stored externally).
- Typically short-lived (bounded by platform timeout limits).
- Commonly used as building blocks for APIs, event handlers, and single-step jobs.
- Retries are common; authors SHOULD design for idempotency when side effects are possible.

Wait states are supported by the platform via execution suspension and later resumption, triggered by events, timers or API callbacks.
For functions this is an advanced capability and is only available for asynchronous execution.

### Workflow

Workflows are durable, multi-step orchestrations that coordinate one or more actions over time.
A workflow is a definition that may contain multiple execution steps, suspension points, and runtime-managed continuation.
- Persisted execution state (durable progress across restarts).
- Supports long-running behavior (timers, waiting on external events, human-in-the-loop patterns).
- Encodes orchestration logic (fan-out/fan-in, branching, retries, compensation) over multiple steps.
- Individual steps are commonly implemented by calling functions or external integrations.

For workflows, the underlying code interpreter/runtime is responsible for typical workflow execution processing, including:
- step identification
- step retry scheduling and retry status
- compensation orchestration
- checkpointing and pause/resume
- event subscription and event-driven continuation

The common Serverless Runtime will provide reusable mechanisms such as an HTTP client with built-in retry mechanisms, registering triggers, subscribing to and emiting events, publishing stauses and checkpoints. The executor for a specific language is responsible for exposing these capabilities in an appropriate way for the language.

Waiting for events, timers or callbacks in workflows is implemented via suspension and trigger registration, with runtime-level examples defined in the code runtime document (e.g. Starlark).

### Functions vs. Workflows

In Hyperspot, functions and workflows use the same entrypoint definition schema and the same implementation language constructs (if/else, loops, variables, etc.).

All synchronous executions are processed as functions. If a workflow explicitly checkpoint during a synchronous request, it is treated as a nil operation. The reason is that with synchronous requests, the onus is on the requester to retry in case of error. Synchronous operations should have very short timeouts, requiring clients to use asynchronous jobs for any long-running operation. Short HTTP timeouts on synchronous requests is a best practice for both client and server performance.

In asynchronous job mode, functions and workflows may OPTIONALLY make use of durable checkpointing provided by core modules (automatic caching of HTTP activities, avoiding re-execution for example) or may explicitly define checkpoint and restore processes within the workflow.

Any function may become a workflow by using one or more of the following runtime-provided facilities:
- Status reporting: provide information about internal progress within the function
- Checkpointing: explicitly record domain-specific information durably for restart.
- Wait states: make a checkpoint, then register a restart trigger. 

Workflow capabilities will be silently ignored when a workflow is executed synchronously, unless a trait specifies the function as async-only - in which case the execution will fail.

Wait states will always result in an explicit error in a synchronous request.

In practice, everything is defined and invoked as a function entrypoint, and the selected runtime MAY determine that a given entrypoint should be treated as a workflow by analyzing the code during validation. For example, the Starlark runtime can validate the program and detect runtime durability primitives (e.g., checkpoints/snapshots or awaiting events/timers); if present, the runtime can apply workflow execution semantics (durable pause/resume and continuation) rather than treating it as a regular short-lived function.
The practical difference is execution semantics:
- Workflows are typically **async** and include internal snapshot/checkpoint behavior so the runtime can persist progress and support pause/resume.
- Functions are typically **sync**, short-lived, and stateless with respect to the runtime.

## Synchronous vs Asynchronous

Functions and workflows can be executed synchronously or asynchronously.

- **sync**
  - The client waits for completion and receives the result (or error) in the HTTP response.
  - This is constrained by HTTP and gateway timeouts, so it is best for short executions.

- **async**
  - The client receives an `job_id` immediately and retrieves status/results later via the status endpoint.
  - The client “polls” for completion:
    - **Short poll**: `GET` returns immediately with the latest known status.
    - **Long poll**: `GET` waits up to a timeout before returning (otherwise identical semantics); this reduces client round-trips while still using polling.

### Streaming

Functions may have a long-running mode where information is streamed to/from a client or another service.  These functions are a category of asynchronous execution, receive a `job_id` that can be referenced later.  This enables them to make use of checkpointing and even restart capabilities.  Durable streams are used to allow a function to reconnect later.

## Functions & Workflows identifier conventions

Both functions and workflows use the same schema and semantic definition following `https://github.com/GlobalTypeSystem/gts-spec` specification.

According to GTS specification, function/workflow contract definitions are represented as GTS **types** (JSON Schema documents), identified by `$id` values that end with `~`.

* Base function/workflow definition:
  - `gts.x.core.faas.func.v1~` - defines the base function/workflow definition schema, including placeholders for `params`, `returns`, `errors`, and the schema for `traits` (semantics) such as timeouts and limits.

* Specific function/workflow definitions:
  - are derived GTS types (JSON Schemas) that reference the base type (typically via `allOf`) and then pin concrete `params`, `returns`, and initialized `traits` values.

Functions/workflows identifier examples:
  - `gts.x.core.faas.func.v1~vendor_x.app.namespace.func_name.v1~`
  - `gts.x.core.faas.func.v1~vendor_y.pkg.domain.workflow_name.v1~`

The invocation API uses GTS ID to refer to the specific function/workflow definition type identifier (the `$id` value without the `gts://` prefix).

## Schemas
This section defines base schema for functions/workflows definitions.

### Base schema: Entrypoint Definition
**Schema ID (type):** `gts.x.core.faas.func.v1~`

Main schema elements are:
- `id` - base function/workflow type identifier.
- `params` - function input schema, to be refined by specific functions/workflows.
- `returns` - function output schema, to be refined by specific functions/workflows.
- `errors` - list of possible custom error type identifiers (GTS IDs).
- `traits` - schema for `traits` (semantics) such as timeouts and limits, that must be pinned by specific functions/workflows (e.g., with property-level `const`)

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "gts://gts.x.core.faas.func.v1~",
  "title": "FaaS Function Definition",
  "type": "object",
  "additionalProperties": false,
  "properties": {
    "id": {
      "type": "string",
      "description": "Well-known type identifier of this function/workflow definition.",
      "examples": ["gts.x.core.faas.func.v1~vendor.app.namespace.func_name.v1~"]
    },

    "title": { "type": "string" },
    "description": { "type": "string" },

    "params": {
      "description": "Function input schema (DTS-style naming). May be a $ref to a GTS schema, inline JSON Schema, or void.",
      "oneOf": [
        {"$ref": "https://json-schema.org/draft/2020-12/schema"},
        {"type": "object", "additionalProperties": true},
        {"type": "null", "description": "Void params"}
      ]
    },

    "returns": {
      "description": "Function output schema (DTS-style naming). May be a $ref to a GTS schema, inline JSON Schema, or void.",
      "oneOf": [
        {"$ref": "https://json-schema.org/draft/2020-12/schema"},
        {"type": "object", "additionalProperties": true},
        {"type": "null", "description": "Void returns"}
      ]
    },

    "errors": {
      "type": "array",
      "items": {"type": "string"},
      "default": [],
      "description": "List of possible customerror type identifiers (GTS IDs)."
    },

    "traits": {
      "type": "object",
      "additionalProperties": false,
      "properties": {
        "entrypoint": {
          "type": "boolean",
          "default": true,
          "description": "If true, the function is externally callable via the invocation API. If false, it is internal-only and can be referenced only from other workflows/functions."
        },
        "is_idempotent": {
          "type": "boolean",
          "default": false,
          "description": "Whether repeated execution with the same effective input produces the same effect."
        },

        "invocation": {
          "type": "object",
          "additionalProperties": false,
          "properties": {
            "supported": {
              "type": "array",
              "items": {"type": "string", "enum": ["sync", "async"]},
              "minItems": 1,
              "uniqueItems": true,
              "default": ["async"],
              "description": "Which invocation modes this entrypoint supports."
            },
            "default": {
              "type": "string",
              "enum": ["sync", "async"],
              "default": "async",
              "description": "Default invocation mode if the caller does not specify one."
            }
          },
          "required": ["supported"]
        },

        "caching": {
          "type": "object",
          "additionalProperties": false,
          "description": "Client caching policy for successful results. Cache reuse is ALWAYS scoped by tenant, caller token/identity, all request headers, and the full `params` payload. This trait is a hint for the caller: the function owner is responsible for providing a reasonable hint, and the runtime cannot guarantee that a cached value equals what the function would return now. This trait only defines freshness TTL.",
          "properties": {
            "max_age_seconds": {
              "type": "integer",
              "minimum": 0,
              "default": 0,
              "description": "How long a cached result is considered fresh (TTL). 0 means no caching (clients MUST NOT reuse cached results). TTL is a client-side reuse hint and does not guarantee correctness under non-determinism, data drift, or changing upstream dependencies."
            }
          },
          "required": ["max_age_seconds"]
        },
        "runtime": {
          "type": "string",
          "description": "Mandatory runtime selector (e.g., Starlark, WASM, AWS Lambda, etc.)."
        },

        "default_timeout_seconds": {
          "type": "integer",
          "minimum": 1,
          "description": "Default timeout for this function/workflow in seconds."
        },
        "default_memory_mb": {
          "type": "integer",
          "minimum": 1,
          "description": "Default memory allocation for this function/workflow in MB."
        },
        "default_cpu": {
          "type": "number",
          "minimum": 0,
          "description": "Default CPU allocation for this function/workflow."
        },

        "localization_dictionary_id": {
          "type": "string",
          "description": "Optional localization dictionary reference (GTS ID)."
        },

        "tags": {
          "type": "array",
          "items": {"type": "string"},
          "default": [],
          "description": "Optional tags for this function/workflow categorization, searchability, etc."
        }

      },
      "required": ["runtime"]
    },

    "implementation": {
      "type": "object",
      "additionalProperties": false,
      "properties": {
        "code": {
          "type": "object",
          "additionalProperties": false,
          "properties": {
            "language": {"type": "string"},
            "source": {
              "type": "string",
              "description": "Inline source code (optional)."
            }
          },
          "required": ["language"]
        }
      },
      "description": "Execution implementation. `code` MUST be provided.",
      "minProperties": 1,
      "required": ["code"]
    }
  },
  "required": ["id", "params", "returns", "traits", "implementation"]
}
```

### Base schema: Error
**Schema ID (type):** `gts.x.core.faas.err.v1~`

This is the base error envelope returned by the runtime (and referenced by entrypoints via `errors`). Derived error types specialize the `details` field.

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "gts://gts.x.core.faas.err.v1~",
  "title": "FaaS Error",
  "type": "object",
  "additionalProperties": false,
  "properties": {
    "id": {
      "type": "string",
      "description": "Error code identifier (GTS ID).",
      "examples": ["gts.x.core.faas.err.v1~x.core._.code.v1~"]
    },
    "message": {
      "type": "string",
      "description": "Human-readable error message."
    },
    "details": {
      "type": "object",
      "additionalProperties": true,
      "description": "Error-class-specific structured payload."
    }
  },
  "required": ["id", "message", "details"]
}
```

### Derived error schema: Upstream HTTP error
**Schema ID (type):** `gts.x.core.faas.err.v1~x.core._.http.v1~`

Used for failures from upstream HTTP calls (e.g., `r_http_get_v1()`). This schema specializes `details`.

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "gts://gts.x.core.faas.err.v1~x.core._.http.v1~",
  "title": "FaaS Upstream HTTP Error",
  "allOf": [
    {"$ref": "gts://gts.x.core.faas.err.v1~"}
  ],
  "type": "object",
  "properties": {
    "id": {"const": "gts.x.core.faas.err.v1~x.core._.http.v1~"},
    "details": {
      "type": "object",
      "additionalProperties": false,
      "properties": {
        "status_code": {"type": "integer", "minimum": 100, "maximum": 599},
        "headers": {
          "type": "object",
          "additionalProperties": {
            "oneOf": [
              {"type": "string"},
              {"type": "array", "items": {"type": "string"}}
            ]
          }
        },
        "body": {
          "type": "object",
          "additionalProperties": true,
          "description": "Parsed JSON body when available; otherwise an empty object."
        }
      },
      "required": ["status_code", "headers", "body"]
    }
  },
  "required": ["id", "message", "details"]
}
```

### Derived error schema: Upstream HTTP transport error (timeout / no connection)
**Schema ID (type):** `gts.x.core.faas.err.v1~x.core._.http_transport.v1~`

Used for upstream HTTP call failures where **no HTTP response** is available (e.g., timeout, connection failure). This schema specializes `details`.

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "gts://gts.x.core.faas.err.v1~x.core._.http_transport.v1~",
  "title": "FaaS Upstream HTTP Transport Error",
  "allOf": [
    {"$ref": "gts://gts.x.core.faas.err.v1~"}
  ],
  "type": "object",
  "properties": {
    "id": {"const": "gts.x.core.faas.err.v1~x.core._.http_transport.v1~"},
    "details": {
      "type": "object",
      "additionalProperties": false,
      "properties": {
        "url": {"type": "string"}
      },
      "required": ["url"]
    }
  },
  "required": ["id", "message", "details"]
}
```

### Derived error schema: Upstream HTTP transport timeout
**Schema ID (type):** `gts.x.core.faas.err.v1~x.core._.http_transport.v1~x.core._.timeout.v1~`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "gts://gts.x.core.faas.err.v1~x.core._.http_transport.v1~x.core._.timeout.v1~",
  "title": "FaaS Upstream HTTP Transport Timeout",
  "allOf": [
    {"$ref": "gts://gts.x.core.faas.err.v1~x.core._.http_transport.v1~"}
  ],
  "type": "object",
  "properties": {
    "id": {"const": "gts.x.core.faas.err.v1~x.core._.http_transport.v1~x.core._.timeout.v1~"}
  },
  "required": ["id", "message", "details"]
}
```

### Derived error schema: Upstream HTTP transport no connection
**Schema ID (type):** `gts.x.core.faas.err.v1~x.core._.http_transport.v1~x.core._.no_connection.v1~`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "gts://gts.x.core.faas.err.v1~x.core._.http_transport.v1~x.core._.no_connection.v1~",
  "title": "FaaS Upstream HTTP Transport No Connection",
  "allOf": [
    {"$ref": "gts://gts.x.core.faas.err.v1~x.core._.http_transport.v1~"}
  ],
  "type": "object",
  "properties": {
    "id": {"const": "gts.x.core.faas.err.v1~x.core._.http_transport.v1~x.core._.no_connection.v1~"}
  },
  "required": ["id", "message", "details"]
}
```

### Derived error schema: Runtime resource/timeout error
**Schema ID (type):** `gts.x.core.faas.err.v1~x.core._.runtime.v1~`

Used when the runtime aborts execution due to resource limits or platform constraints.

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "gts://gts.x.core.faas.err.v1~x.core._.runtime.v1~",
  "title": "FaaS Runtime Error",
  "allOf": [
    {"$ref": "gts://gts.x.core.faas.err.v1~"}
  ],
  "type": "object",
  "properties": {
    "id": {"const": "gts.x.core.faas.err.v1~x.core._.runtime.v1~"},
    "details": {
      "type": "object",
      "additionalProperties": false,
      "properties": {
        "limit": {
          "type": "object",
          "additionalProperties": false,
          "properties": {
            "timeout_seconds": {"type": "integer", "minimum": 1},
            "memory_limit_mb": {"type": "integer", "minimum": 1},
            "cpu_limit": {"type": "number", "minimum": 0}
          }
        },
        "observed": {
          "type": "object",
          "additionalProperties": false,
          "properties": {
            "duration_ms": {"type": "integer", "minimum": 0},
            "cpu_time_ms": {"type": "integer", "minimum": 0},
            "max_memory_used_mb": {"type": "integer", "minimum": 0}
          }
        }
      },
      "required": []
    }
  },
  "required": ["id", "message", "details"]
}
```

### Derived error schema: Runtime timeout
**Schema ID (type):** `gts.x.core.faas.err.v1~x.core._.runtime.v1~x.core._.timeout.v1~`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "gts://gts.x.core.faas.err.v1~x.core._.runtime.v1~x.core._.timeout.v1~",
  "title": "FaaS Runtime Timeout",
  "allOf": [
    {"$ref": "gts://gts.x.core.faas.err.v1~x.core._.runtime.v1~"}
  ],
  "type": "object",
  "properties": {
    "id": {"const": "gts.x.core.faas.err.v1~x.core._.runtime.v1~x.core._.timeout.v1~"}
  },
  "required": ["id", "message", "details"]
}
```

### Derived error schema: Runtime memory limit
**Schema ID (type):** `gts.x.core.faas.err.v1~x.core._.runtime.v1~x.core._.memory_limit.v1~`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "gts://gts.x.core.faas.err.v1~x.core._.runtime.v1~x.core._.memory_limit.v1~",
  "title": "FaaS Runtime Memory Limit",
  "allOf": [
    {"$ref": "gts://gts.x.core.faas.err.v1~x.core._.runtime.v1~"}
  ],
  "type": "object",
  "properties": {
    "id": {"const": "gts.x.core.faas.err.v1~x.core._.runtime.v1~x.core._.memory_limit.v1~"}
  },
  "required": ["id", "message", "details"]
}
```

### Derived error schema: Runtime CPU limit
**Schema ID (type):** `gts.x.core.faas.err.v1~x.core._.runtime.v1~x.core._.cpu_limit.v1~`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "gts://gts.x.core.faas.err.v1~x.core._.runtime.v1~x.core._.cpu_limit.v1~",
  "title": "FaaS Runtime CPU Limit",
  "allOf": [
    {"$ref": "gts://gts.x.core.faas.err.v1~x.core._.runtime.v1~"}
  ],
  "type": "object",
  "properties": {
    "id": {"const": "gts.x.core.faas.err.v1~x.core._.runtime.v1~x.core._.cpu_limit.v1~"}
  },
  "required": ["id", "message", "details"]
}
```

### Derived error schema: Runtime canceled
**Schema ID (type):** `gts.x.core.faas.err.v1~x.core._.runtime.v1~x.core._.canceled.v1~`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "gts://gts.x.core.faas.err.v1~x.core._.runtime.v1~x.core._.canceled.v1~",
  "title": "FaaS Runtime Canceled",
  "allOf": [
    {"$ref": "gts://gts.x.core.faas.err.v1~x.core._.runtime.v1~"}
  ],
  "type": "object",
  "properties": {
    "id": {"const": "gts.x.core.faas.err.v1~x.core._.runtime.v1~x.core._.canceled.v1~"}
  },
  "required": ["id", "message", "details"]
}
```

### Derived error schema: Code execution/validation error
**Schema ID (type):** `gts.x.core.faas.err.v1~x.core._.code.v1~`

Used for code-level failures produced by the selected runtime (e.g., Starlark validation/compile/runtime errors).

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "gts://gts.x.core.faas.err.v1~x.core._.code.v1~",
  "title": "FaaS Code Error",
  "allOf": [
    {"$ref": "gts://gts.x.core.faas.err.v1~"}
  ],
  "type": "object",
  "properties": {
    "id": {"const": "gts.x.core.faas.err.v1~x.core._.code.v1~"},
    "details": {
      "type": "object",
      "additionalProperties": false,
      "properties": {
        "runtime": {"type": "string", "description": "Language/runtime identifier (e.g., starlark)."},
        "phase": {"type": "string", "enum": ["validate", "compile", "execute"]},
        "error_kind": {"type": "string", "description": "Runtime-specific diagnostic kind/code (not an exception)."},
        "location": {
          "type": "object",
          "additionalProperties": false,
          "properties": {
            "line": {"type": "integer", "minimum": 1},
            "code": {"type": "string"}
          },
          "required": ["line", "code"]
        },
        "stack": {
          "type": "object",
          "additionalProperties": false,
          "properties": {
            "frames": {
              "type": "array",
              "items": {
                "type": "object",
                "additionalProperties": false,
                "properties": {
                  "function": {"type": "string"},
                  "file": {"type": "string"},
                  "line": {"type": "integer", "minimum": 1}
                },
                "required": ["function", "file", "line"]
              }
            }
          },
          "required": ["frames"]
        }
      },
      "required": ["runtime", "phase"]
    }
  },
  "required": ["id", "message", "details"]
}
```

### Execution status identifiers

Execution status is represented as a GTS type identifier. The base status type is `gts.x.core.faas.status.v1~`, with derived types for each status value:

| Status ID | Description |
|-----------|-------------|
| `gts.x.core.faas.status.v1~x.core._.queued.v1~` | Job accepted, waiting to start execution. |
| `gts.x.core.faas.status.v1~x.core._.running.v1~` | Job is currently executing. |
| `gts.x.core.faas.status.v1~x.core._.paused.v1~` | Execution paused by user request or waiting on a timer/external event. Can be resumed manually or by the runtime. |
| `gts.x.core.faas.status.v1~x.core._.succeeded.v1~` | Execution completed successfully. Result available. |
| `gts.x.core.faas.status.v1~x.core._.failed.v1~` | Execution failed. Error details available. |
| `gts.x.core.faas.status.v1~x.core._.canceled.v1~` | Execution was canceled by request. |

## Examples
### Function definition type example #1

This is a tax calculation function example written in Starlark that:
- Supports both sync and async invocation (default is async)
- Idempotent
- Includes default resource/time limits
- Enables client caching for 60 seconds (within tenant + token/identity + headers + params)

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "gts://gts.x.core.faas.func.v1~vendor.app.billing.calculate_tax.v1~",
  "title": "Calculate Tax",
  "allOf": [
    {"$ref": "gts://gts.x.core.faas.func.v1~"}
  ],
  "type": "object",
  "properties": {
    "id": {"const": "gts.x.core.faas.func.v1~vendor.app.billing.calculate_tax.v1~"},
    "params": {
      "const": {
        "type": "object",
        "additionalProperties": false,
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
        "additionalProperties": false,
        "properties": {
          "tax": {"type": "number"}
        },
        "required": ["tax"]
      }
    },
    "errors": {"const": ["gts.x.core.faas.err.v1~x.core._.code.v1~", "gts.x.core.faas.err.v1~vendor.app.billing.tax_error.v1~"]},
    "traits": {
      "type": "object",
      "properties": {
        "runtime": {"const": "starlark"},
        "is_idempotent": {"const": true},
        "invocation": {
          "type": "object",
          "properties": {
            "supported": {"const": ["sync", "async"]},
            "default": {"const": "async"}
          }
        },
        "default_timeout_seconds": {"const": 10},
        "default_memory_mb": {"const": 128},
        "caching": {
          "type": "object",
          "properties": {
            "max_age_seconds": {"const": 60}
          },
          "required": ["max_age_seconds"]
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
            "source": {"const": "def main(ctx, input):\n  return {\"tax\": 0}\n"}
          },
          "required": ["language"]
        }
      },
      "required": ["code"]
    }
  },
  "required": ["id", "params", "returns", "traits", "implementation"]
}
```

### Function definition type example #2

This is an example of a customer lookup function written in Starlark that:
- Supports both sync and async invocation (default is sync)
- Idempotent
- No client caching by default (`max_age_seconds = 0`)

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "gts://gts.x.core.faas.func.v1~vendor.app.crm.lookup_customer.v1~",
  "allOf": [
    {"$ref": "gts://gts.x.core.faas.func.v1~"}
  ],
  "type": "object",
  "properties": {
    "id": {"const": "gts.x.core.faas.func.v1~vendor.app.crm.lookup_customer.v1~"},
    "params": {
      "const": {
        "type": "object",
        "additionalProperties": false,
        "properties": {
          "customer_id": {"type": "string"}
        },
        "required": ["customer_id"]
      }
    },
    "returns": {
      "const": {
        "type": "object",
        "additionalProperties": false,
        "properties": {
          "customer": {
            "type": "object",
            "additionalProperties": true
          }
        },
        "required": ["customer"]
      }
    },
    "traits": {
      "type": "object",
      "properties": {
        "runtime": {"const": "starlark"},
        "is_idempotent": {"const": true},
        "invocation": {
          "type": "object",
          "properties": {
            "supported": {"const": ["sync", "async"]},
            "default": {"const": "sync"}
          }
        },
        "default_timeout_seconds": {"const": 5},
        "caching": {
          "type": "object",
          "properties": {
            "max_age_seconds": {"const": 0}
          },
          "required": ["max_age_seconds"]
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
            "source": {"const": "def main(ctx, input):\\n  url = \"https://example.crm/api/customer/\" + input[\"customer_id\"]\\n  resp = r_http_get_v1(url)\\n  return {\"customer\": resp}\\n"}
          },
          "required": ["language"]
        }
      },
      "required": ["code"]
    }
  },
  "required": ["id", "params", "returns", "traits", "implementation"]
}
```

## Invocation API (Entrypoint)

The Serverless Runtime exposes two invocation surfaces:
1. **JSON-RPC 2.0** — Synchronous request/response execution via a standard JSON-RPC endpoint.
2. **Async Jobs API** — Asynchronous execution that mirrors the JSON-RPC request format but returns a job ID for later polling.

### Principles
- Synchronous invocations use standard JSON-RPC 2.0 semantics.
- Asynchronous invocations produce a **job record** with a stable `job_id`.
- All invocations SHOULD support:
  - `Idempotency-Key` header (to prevent duplicate starts)
  - `X-Request-Id` / correlation id header
  - `traceparent` header (W3C trace context)

Throttling and rate limiting:
- The runtime SHOULD enforce throttling to protect itself and downstream dependencies (e.g., per-tenant rate limits, per-entrypoint concurrency caps).
- When throttled, the runtime SHOULD return HTTP `429 Too Many Requests` and SHOULD include `Retry-After`.

Dry run:
- `dry_run: true` in params performs validation (including authorization) without execution or creating a durable job record.
- This aligns with AWS Lambda `InvocationType=DryRun`.

---

## JSON-RPC API (Synchronous)

### Endpoint
`POST /api/serverless-runtime/v1/rpc`

All synchronous invocations use JSON-RPC 2.0 over HTTP POST. The method name is the entrypoint GTS ID.

### Request format
```json
{
  "jsonrpc": "2.0",
  "method": "gts.x.core.faas.func.v1~vendor.app.namespace.func_name.v1~",
  "params": {
    "invoice_total": 100.00,
    "region": "US-CA"
  },
  "id": "req-12345"
}
```

| Field | Type | Description |
|-------|------|-------------|
| `jsonrpc` | string | Must be `"2.0"`. |
| `method` | string | The entrypoint GTS ID to invoke. |
| `params` | object | Input parameters matching the entrypoint's `params` schema. May include `dry_run: true` to validate without executing. |
| `id` | string \| number | Client-provided request identifier. Returned in response. |

### Success response
`200 OK`
```json
{
  "jsonrpc": "2.0",
  "result": {
    "tax": 7.25
  },
  "id": "req-12345"
}
```

### Error response (entrypoint execution error)
`200 OK`
```json
{
  "jsonrpc": "2.0",
  "error": {
    "code": -32000,
    "message": "Upstream CRM returned non-success status",
    "data": {
      "id": "gts.x.core.faas.err.v1~x.core._.http.v1~",
      "details": {
        "status_code": 503,
        "headers": {"content-type": "application/json"},
        "body": {"error": "service_unavailable"}
      },
      "observability": {
        "correlation_id": "<id>",
        "trace_id": "<trace>",
        "metrics": {
          "duration_ms": 110,
          "billed_duration_ms": 200,
          "cpu_time_ms": 70,
          "memory_limit_mb": 128,
          "max_memory_used_mb": 64
        }
      }
    }
  },
  "id": "req-12345"
}
```

### Error response (code execution error)
`200 OK`
```json
{
  "jsonrpc": "2.0",
  "error": {
    "code": -32000,
    "message": "Starlark execution error: division by zero",
    "data": {
      "id": "gts.x.core.faas.err.v1~x.core._.code.v1~",
      "details": {
        "runtime": "starlark",
        "phase": "execute",
        "error_kind": "division_by_zero",
        "location": {"line": 12, "code": "x = 1 / 0"},
        "stack": {
          "frames": [
            {"function": "main", "file": "inline", "line": 1},
            {"function": "calculate", "file": "inline", "line": 12}
          ]
        }
      },
      "observability": {
        "correlation_id": "<id>",
        "trace_id": "<trace>",
        "metrics": {
          "duration_ms": 10,
          "billed_duration_ms": 100,
          "cpu_time_ms": 8,
          "memory_limit_mb": 128,
          "max_memory_used_mb": 24
        }
      }
    }
  },
  "id": "req-12345"
}
```

### Error response (runtime timeout)
`200 OK`
```json
{
  "jsonrpc": "2.0",
  "error": {
    "code": -32000,
    "message": "Execution timed out",
    "data": {
      "id": "gts.x.core.faas.err.v1~x.core._.runtime.v1~x.core._.timeout.v1~",
      "details": {
        "limit": {"timeout_seconds": 5},
        "observed": {"duration_ms": 5000, "cpu_time_ms": 4900, "max_memory_used_mb": 90}
      },
      "observability": {
        "correlation_id": "<id>",
        "trace_id": "<trace>",
        "metrics": {
          "duration_ms": 5000,
          "billed_duration_ms": 5000,
          "cpu_time_ms": 4900,
          "memory_limit_mb": 128,
          "max_memory_used_mb": 90
        }
      }
    }
  },
  "id": "req-12345"
}
```

### Standard JSON-RPC error codes
| Code | Message | Description |
|------|---------|-------------|
| `-32700` | Parse error | Invalid JSON was received. |
| `-32600` | Invalid Request | The JSON sent is not a valid Request object. |
| `-32601` | Method not found | The entrypoint does not exist or is not callable. |
| `-32602` | Invalid params | Invalid method parameter(s) — schema validation failed. |
| `-32603` | Internal error | Internal JSON-RPC error. |
| `-32000` | Server error | Application-level error (entrypoint execution failure). Details in `data`. |

### Dry run response
When `params` includes `dry_run: true`:
`200 OK`
```json
{
  "jsonrpc": "2.0",
  "result": {
    "dry_run": true,
    "valid": true
  },
  "id": "req-12345"
}
```

### Batch requests
JSON-RPC 2.0 batch requests are supported. Send an array of request objects:
```json
[
  {"jsonrpc": "2.0", "method": "gts.x.core.faas.func.v1~vendor.app.billing.calculate_tax.v1~", "params": {"invoice_total": 100, "region": "US-CA"}, "id": "1"},
  {"jsonrpc": "2.0", "method": "gts.x.core.faas.func.v1~vendor.app.crm.lookup_customer.v1~", "params": {"customer_id": "c_123"}, "id": "2"}
]
```

Response is an array of response objects (order may differ):
```json
[
  {"jsonrpc": "2.0", "result": {"tax": 7.25}, "id": "1"},
  {"jsonrpc": "2.0", "result": {"customer": {"id": "c_123", "name": "Jane"}}, "id": "2"}
]
```

---

## Async Jobs API

The Async Jobs API mirrors the JSON-RPC request format but immediately returns a job ID instead of waiting for completion. Use this for long-running operations or when you want fire-and-forget semantics with optional polling.

### 1) Submit async job
`POST /api/serverless-runtime/v1/jobs`

Request body (mirrors JSON-RPC structure):
```json
{
  "jsonrpc": "2.0",
  "method": "gts.x.core.faas.func.v1~vendor.app.namespace.func_name.v1~",
  "params": {
    "invoice_total": 100.00,
    "region": "US-CA"
  },
  "id": "client-correlation-123"
}
```

Response:
`202 Accepted`
```json
{
  "job_id": "<opaque-job-id>",
  "method": "gts.x.core.faas.func.v1~vendor.app.namespace.func_name.v1~",
  "status": "gts.x.core.faas.status.v1~x.core._.queued.v1~",
  "client_id": "client-correlation-123",
  "timestamps": {
    "created_at": "2026-01-01T00:00:00.000Z",
    "started_at": null,
    "finished_at": null
  },
  "result": null,
  "error": null,
  "observability": {
    "correlation_id": "<id>",
    "trace_id": "<trace>",
    "metrics": null
  }
}
```

### 2) List jobs
`GET /api/serverless-runtime/v1/jobs`

Returns a paginated list of jobs matching the specified filters. Supports filtering by status, time range, entrypoint, and correlation identifier.

Query params:
| Parameter | Type | Description |
|-----------|------|-------------|
| `status` | string | Filter by status GTS ID (e.g., `gts.x.core.faas.status.v1~x.core._.running.v1~`). Supports multiple values comma-separated. |
| `method` | string | Filter by entrypoint GTS ID. |
| `created_after` | string (ISO 8601) | Return jobs created at or after this timestamp. |
| `created_before` | string (ISO 8601) | Return jobs created before this timestamp. |
| `correlation_id` | string | Filter by correlation identifier. |
| `client_id` | string | Filter by client-provided request ID. |
| `limit` | integer | Maximum number of results (default: 25, max: 100). |
| `cursor` | string | Pagination cursor from previous response. |

Response:
`200 OK`
```json
{
  "items": [
    {
      "job_id": "<opaque-job-id-1>",
      "method": "gts.x.core.faas.func.v1~vendor.app.billing.calculate_tax.v1~",
      "status": "gts.x.core.faas.status.v1~x.core._.succeeded.v1~",
      "client_id": "client-correlation-123",
      "timestamps": {
        "created_at": "2026-01-01T00:00:00.000Z",
        "started_at": "2026-01-01T00:00:00.010Z",
        "finished_at": "2026-01-01T00:00:00.120Z"
      },
      "observability": {
        "correlation_id": "<id>",
        "trace_id": "<trace>"
      }
    },
    {
      "job_id": "<opaque-job-id-2>",
      "method": "gts.x.core.faas.func.v1~vendor.app.crm.lookup_customer.v1~",
      "status": "gts.x.core.faas.status.v1~x.core._.running.v1~",
      "client_id": "client-correlation-456",
      "timestamps": {
        "created_at": "2026-01-01T00:00:01.000Z",
        "started_at": "2026-01-01T00:00:01.010Z",
        "finished_at": null
      },
      "observability": {
        "correlation_id": "<id>",
        "trace_id": "<trace>"
      }
    }
  ],
  "page_info": {
    "next_cursor": "<opaque-cursor>",
    "prev_cursor": null,
    "limit": 25,
    "total_count": 142
  }
}
```

Note: The list response includes summary information for each job. To retrieve full job details including `result`, `error`, and `observability.metrics`, use `GET /jobs/{job_id}`.

### 3) Get job status
`GET /api/serverless-runtime/v1/jobs/{job_id}`

Response — **Job queued**:
`200 OK`
```json
{
  "job_id": "<opaque-job-id>",
  "method": "gts.x.core.faas.func.v1~vendor.app.namespace.func_name.v1~",
  "status": "gts.x.core.faas.status.v1~x.core._.queued.v1~",
  "client_id": "client-correlation-123",
  "timestamps": {
    "created_at": "2026-01-01T00:00:00.000Z",
    "started_at": null,
    "finished_at": null
  },
  "result": null,
  "error": null,
  "observability": {
    "correlation_id": "<id>",
    "trace_id": "<trace>",
    "metrics": null
  }
}
```

Response — **Job running**:
`200 OK`
```json
{
  "job_id": "<opaque-job-id>",
  "method": "gts.x.core.faas.func.v1~vendor.app.namespace.func_name.v1~",
  "status": "gts.x.core.faas.status.v1~x.core._.running.v1~",
  "client_id": "client-correlation-123",
  "timestamps": {
    "created_at": "2026-01-01T00:00:00.000Z",
    "started_at": "2026-01-01T00:00:00.010Z",
    "finished_at": null
  },
  "result": null,
  "error": null,
  "observability": {
    "correlation_id": "<id>",
    "trace_id": "<trace>",
    "metrics": {
      "duration_ms": null,
      "billed_duration_ms": null,
      "cpu_time_ms": null,
      "memory_limit_mb": 128,
      "max_memory_used_mb": null
    }
  }
}
```

Response — **Job succeeded**:
`200 OK`
```json
{
  "job_id": "<opaque-job-id>",
  "method": "gts.x.core.faas.func.v1~vendor.app.namespace.func_name.v1~",
  "status": "gts.x.core.faas.status.v1~x.core._.succeeded.v1~",
  "client_id": "client-correlation-123",
  "timestamps": {
    "created_at": "2026-01-01T00:00:00.000Z",
    "started_at": "2026-01-01T00:00:00.010Z",
    "finished_at": "2026-01-01T00:00:00.120Z"
  },
  "result": {
    "tax": 7.25
  },
  "error": null,
  "observability": {
    "correlation_id": "<id>",
    "trace_id": "<trace>",
    "metrics": {
      "duration_ms": 110,
      "billed_duration_ms": 200,
      "cpu_time_ms": 70,
      "memory_limit_mb": 128,
      "max_memory_used_mb": 64
    }
  }
}
```

Response — **Job failed (code error)**:
`200 OK`
```json
{
  "job_id": "<opaque-job-id>",
  "method": "gts.x.core.faas.func.v1~vendor.app.billing.calculate_tax.v1~",
  "status": "gts.x.core.faas.status.v1~x.core._.failed.v1~",
  "client_id": "client-correlation-123",
  "timestamps": {
    "created_at": "2026-01-01T00:00:00.000Z",
    "started_at": "2026-01-01T00:00:00.010Z",
    "finished_at": "2026-01-01T00:00:00.020Z"
  },
  "result": null,
  "error": {
    "code": -32000,
    "message": "Starlark execution error: division by zero",
    "data": {
      "id": "gts.x.core.faas.err.v1~x.core._.code.v1~",
      "details": {
        "runtime": "starlark",
        "phase": "execute",
        "error_kind": "division_by_zero",
        "location": {"line": 12, "code": "x = 1 / 0"},
        "stack": {
          "frames": [
            {"function": "main", "file": "inline", "line": 1},
            {"function": "calculate", "file": "inline", "line": 12}
          ]
        }
      }
    }
  },
  "observability": {
    "correlation_id": "<id>",
    "trace_id": "<trace>",
    "metrics": {
      "duration_ms": 10,
      "billed_duration_ms": 100,
      "cpu_time_ms": 8,
      "memory_limit_mb": 128,
      "max_memory_used_mb": 24
    }
  }
}
```

Response — **Job failed (runtime timeout)**:
`200 OK`
```json
{
  "job_id": "<opaque-job-id>",
  "method": "gts.x.core.faas.func.v1~vendor.app.namespace.func_name.v1~",
  "status": "gts.x.core.faas.status.v1~x.core._.failed.v1~",
  "client_id": "client-correlation-123",
  "timestamps": {
    "created_at": "2026-01-01T00:00:00.000Z",
    "started_at": "2026-01-01T00:00:00.010Z",
    "finished_at": "2026-01-01T00:00:05.010Z"
  },
  "result": null,
  "error": {
    "code": -32000,
    "message": "Execution timed out",
    "data": {
      "id": "gts.x.core.faas.err.v1~x.core._.runtime.v1~x.core._.timeout.v1~",
      "details": {
        "limit": {"timeout_seconds": 5},
        "observed": {"duration_ms": 5000, "cpu_time_ms": 4900, "max_memory_used_mb": 90}
      }
    }
  },
  "observability": {
    "correlation_id": "<id>",
    "trace_id": "<trace>",
    "metrics": {
      "duration_ms": 5000,
      "billed_duration_ms": 5000,
      "cpu_time_ms": 4900,
      "memory_limit_mb": 128,
      "max_memory_used_mb": 90
    }
  }
}
```

Response — **Job failed (upstream no connection)**:
`200 OK`
```json
{
  "job_id": "<opaque-job-id>",
  "method": "gts.x.core.faas.func.v1~vendor.app.crm.lookup_customer.v1~",
  "status": "gts.x.core.faas.status.v1~x.core._.failed.v1~",
  "client_id": "client-correlation-123",
  "timestamps": {
    "created_at": "2026-01-01T00:00:00.000Z",
    "started_at": "2026-01-01T00:00:00.010Z",
    "finished_at": "2026-01-01T00:00:00.520Z"
  },
  "result": null,
  "error": {
    "code": -32000,
    "message": "Upstream HTTP request failed",
    "data": {
      "id": "gts.x.core.faas.err.v1~x.core._.http_transport.v1~x.core._.no_connection.v1~",
      "details": {
        "url": "https://example.crm/api/customers/123"
      }
    }
  },
  "observability": {
    "correlation_id": "<id>",
    "trace_id": "<trace>",
    "metrics": {
      "duration_ms": 510,
      "billed_duration_ms": 600,
      "cpu_time_ms": 30,
      "memory_limit_mb": 128,
      "max_memory_used_mb": 40
    }
  }
}
```

Response — **Job paused** (waiting on timer/event or user-paused):
`200 OK`
```json
{
  "job_id": "<opaque-job-id>",
  "method": "gts.x.core.faas.func.v1~vendor.app.namespace.func_name.v1~",
  "status": "gts.x.core.faas.status.v1~x.core._.paused.v1~",
  "client_id": "client-correlation-123",
  "timestamps": {
    "created_at": "2026-01-01T00:00:00.000Z",
    "started_at": "2026-01-01T00:00:00.010Z",
    "finished_at": null
  },
  "result": null,
  "error": null,
  "observability": {
    "correlation_id": "<id>",
    "trace_id": "<trace>",
    "metrics": {
      "duration_ms": 50,
      "billed_duration_ms": 100,
      "cpu_time_ms": 40,
      "memory_limit_mb": 128,
      "max_memory_used_mb": 32
    }
  }
}
```

### 4) Get job result (convenience endpoint)
`GET /api/serverless-runtime/v1/jobs/{job_id}/result`

Returns only the result payload when the job has succeeded. Useful for clients that only care about the final output.

Response — **Job succeeded**:
`200 OK`
```json
{
  "tax": 7.25
}
```

Response — **Job not finished**:
`409 Conflict`
```json
{
  "error": "job_not_finished",
  "message": "Job is still in progress",
  "status": "gts.x.core.faas.status.v1~x.core._.running.v1~"
}
```

Response — **Job failed**:
`200 OK` with JSON-RPC error structure:
```json
{
  "error": {
    "code": -32000,
    "message": "Starlark execution error: division by zero",
    "data": {
      "id": "gts.x.core.faas.err.v1~x.core._.code.v1~",
      "details": {
        "runtime": "starlark",
        "phase": "execute",
        "error_kind": "division_by_zero",
        "location": {"line": 12, "code": "x = 1 / 0"},
        "stack": {
          "frames": [
            {"function": "main", "file": "inline", "line": 1},
            {"function": "calculate", "file": "inline", "line": 12}
          ]
        }
      }
    }
  }
}
```

### 5) Cancel job
`POST /api/serverless-runtime/v1/jobs/{job_id}:cancel`

Requests cancellation of an in-flight job. The runtime will attempt to stop execution gracefully. If the job is not in a cancelable state (e.g., already completed, failed, or canceled), the request returns an error.

Response:
`202 Accepted`
```json
{
  "job_id": "<opaque-job-id>",
  "status": "gts.x.core.faas.status.v1~x.core._.canceled.v1~",
  "message": "Cancellation requested"
}
```

Response — **Job not cancelable**:
`409 Conflict`
```json
{
  "error": "job_not_cancelable",
  "message": "Job cannot be canceled in its current state",
  "status": "gts.x.core.faas.status.v1~x.core._.succeeded.v1~"
}
```

### 6) Replay job
`POST /api/serverless-runtime/v1/jobs/{job_id}:replay`

Creates a new job that re-executes the same entrypoint with the same input parameters as the original job. Useful for debugging, incident analysis, and controlled recovery after failures.

The replay creates a **new job** with a new `job_id`. The original job remains unchanged for audit purposes.

Request body (optional overrides):
```json
{
  "params_override": {
    "region": "US-NY"
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `params_override` | object | (Optional) Partial parameter overrides to merge with the original params. |

Response:
`202 Accepted`
```json
{
  "job_id": "<new-job-id>",
  "replayed_from": "<original-job-id>",
  "method": "gts.x.core.faas.func.v1~vendor.app.namespace.func_name.v1~",
  "status": "gts.x.core.faas.status.v1~x.core._.queued.v1~",
  "client_id": "client-correlation-123",
  "timestamps": {
    "created_at": "2026-01-01T00:00:00.000Z",
    "started_at": null,
    "finished_at": null
  },
  "result": null,
  "error": null,
  "observability": {
    "correlation_id": "<id>",
    "trace_id": "<trace>",
    "metrics": null
  }
}
```

Response — **Original job not found**:
`404 Not Found`
```json
{
  "error": "job_not_found",
  "message": "The specified job does not exist or has been purged.",
  "status": null
}
```

### 7) Pause job
`POST /api/serverless-runtime/v1/jobs/{job_id}:pause`

Requests the runtime to pause an in-flight job at the next safe checkpoint. The job will transition to `paused` status and can be resumed later. If the job is not in a pausable state (e.g., already completed, canceled, or paused), the request returns an error.

Response:
`202 Accepted`
```json
{
  "job_id": "<opaque-job-id>",
  "status": "gts.x.core.faas.status.v1~x.core._.paused.v1~",
  "message": "Pause requested"
}
```

Response — **Job not pausable**:
`409 Conflict`
```json
{
  "error": "job_not_pausable",
  "message": "Job cannot be paused in its current state",
  "status": "gts.x.core.faas.status.v1~x.core._.succeeded.v1~"
}
```

### 8) Resume job
`POST /api/serverless-runtime/v1/jobs/{job_id}:resume`

Resumes a paused job. The job will transition back to `running` status and continue execution from its last checkpoint. If the job is not in `paused` status, the request returns an error.

Response:
`202 Accepted`
```json
{
  "job_id": "<opaque-job-id>",
  "status": "gts.x.core.faas.status.v1~x.core._.running.v1~",
  "message": "Resume requested"
}
```

Response — **Job not paused**:
`409 Conflict`
```json
{
  "error": "job_not_paused",
  "message": "Job is not in paused state",
  "status": "gts.x.core.faas.status.v1~x.core._.running.v1~"
}
```

### Job status values

See [Execution status identifiers](#execution-status-identifiers) for full descriptions.

| Status ID (short form) | GTS ID |
|------------------------|--------|
| `queued` | `gts.x.core.faas.status.v1~x.core._.queued.v1~` |
| `running` | `gts.x.core.faas.status.v1~x.core._.running.v1~` |
| `paused` | `gts.x.core.faas.status.v1~x.core._.paused.v1~` |
| `succeeded` | `gts.x.core.faas.status.v1~x.core._.succeeded.v1~` |
| `failed` | `gts.x.core.faas.status.v1~x.core._.failed.v1~` |
| `canceled` | `gts.x.core.faas.status.v1~x.core._.canceled.v1~` |

---

## Executions API (Observability)

Every execution — whether synchronous (JSON-RPC) or asynchronous (Jobs API) — produces an **execution record** identified by an `execution_id`. This ID can be used to retrieve observability data after execution completes or while it is in progress.

**How to obtain the `execution_id`:**
- **Synchronous (JSON-RPC)**: The `execution_id` is returned in the `X-Execution-Id` response header.
- **Asynchronous (Jobs API)**: The `job_id` returned by the Jobs API is also the `execution_id`, and is returned in the `X-Execution-Id` header.

The Executions API provides read-only access to execution history, debugging information, and execution traces.

### 1) Timeline / step history
`GET /api/serverless-runtime/v1/executions/{execution_id}/timeline`

Query params:
- `limit` (optional)
- `cursor` (optional)

Response (example):
```json
{
  "execution_id": "<opaque-id>",
  "items": [
    {"at": "2026-01-01T00:00:00Z", "type": "STARTED"},
    {"at": "2026-01-01T00:00:01Z", "type": "STEP_STARTED", "step_id": "step-1"},
    {"at": "2026-01-01T00:00:02Z", "type": "STEP_SUCCEEDED", "step_id": "step-1"}
  ],
  "page_info": {
    "next_cursor": "<opaque-cursor>",
    "prev_cursor": null,
    "limit": 25
  }
}
```

### 2) Debug execution
`GET /api/serverless-runtime/v1/executions/{execution_id}/debug`

Response (example):
```json
{
  "execution_id": "<opaque-id>",
  "entrypoint_id": "gts.x.core.faas.func.v1~vendor.app.namespace.func_name.v1~",
  "status": "gts.x.core.faas.status.v1~x.core._.running.v1~",
  "timestamps": {
    "created_at": "2026-01-01T00:00:00.000Z",
    "started_at": "2026-01-01T00:00:00.010Z",
    "finished_at": null
  },
  "result": null,
  "error": null,
  "observability": {
    "correlation_id": "<id>",
    "trace_id": "<trace>",
    "metrics": {
      "duration_ms": null,
      "billed_duration_ms": null,
      "cpu_time_ms": null,
      "memory_limit_mb": 128,
      "max_memory_used_mb": null
    }
  },
  "debug": {
    "location": {
      "line": 12,
      "code": "resp = r_http_get_v1(url)"
    },
    "stack": {
      "frames": [
        {"function": "main", "file": "inline", "line": 1},
        {"function": "calculate", "file": "inline", "line": 12}
      ]
    }
  }
}
```

### 3) Execution trace
`GET /api/serverless-runtime/v1/executions/{execution_id}/trace`

Query params:
- `limit` (optional)
- `cursor` (optional)

Response (example):
```json
{
  "execution_id": "<opaque-id>",
  "entrypoint_id": "gts.x.core.faas.func.v1~vendor.app.namespace.func_name.v1~",
  "status": "gts.x.core.faas.status.v1~x.core._.running.v1~",
  "timestamps": {
    "created_at": "2026-01-01T00:00:00.000Z",
    "started_at": "2026-01-01T00:00:00.010Z",
    "finished_at": null
  },
  "result": null,
  "error": null,
  "observability": {
    "correlation_id": "<id>",
    "trace_id": "<trace>",
    "metrics": {
      "duration_ms": null,
      "billed_duration_ms": null,
      "cpu_time_ms": null,
      "memory_limit_mb": 128,
      "max_memory_used_mb": null
    }
  },
  "debug": {
    "location": {
      "line": 12,
      "code": "resp = r_http_get_v1(url)"
    },
    "stack": {
      "frames": [
        {"function": "main", "file": "inline", "line": 1},
        {"function": "calculate", "file": "inline", "line": 12}
      ]
    },
    "page": {
      "items": [
        {
          "call_execution_id": "<opaque-id>",
          "entrypoint_id": "gts.x.core.faas.func.v1~vendor.app.crm.lookup_customer.v1~",
          "params": {"customer_id": "c_123"},
          "duration_ms": 42,
          "response": {
            "result": {"customer": {"id": "c_123", "name": "Jane"}},
            "error": null
          }
        }
      ],
      "page_info": {
        "next_cursor": "<opaque-cursor>",
        "prev_cursor": null,
        "limit": 25
      }
    }
  }
}
```

## TODO:

## Comparison with similar solutions

This section compares the proposed Serverless Runtime entrypoint model and invocation APIs with similar capabilities in:
- AWS (Lambda + Step Functions)
- Google Cloud (Cloud Functions + Workflows)
- Azure (Azure Functions + Durable Functions)

Notes:
- Public clouds generally split “function” and “workflow/orchestration” into different products.
- Hyperspot deliberately exposes a unified entrypoint definition schema and a single invocation surface, with consistent response shapes.

### Category: Definition model and versioning

| Capability | Hyperspot Serverless Runtime | AWS | Google Cloud | Azure |
|---|---|---|---|---|
| Definition type model | Unified entrypoint definition schema (function/workflow) via GTS-identified JSON Schemas | Split: Lambda functions vs Step Functions state machines | Split: Cloud Functions vs Workflows | Split: Functions vs Durable orchestrations |
| Type system / schema IDs | GTS identifiers for base + derived definition types | No first-class type IDs; resource ARNs + service-specific specs | No first-class type IDs; resource names + service-specific specs | No first-class type IDs; resource IDs + service-specific specs |
| Versioning concept | Definitions are versioned types; executions reference an explicit definition ID | Lambda versions/aliases; Step Functions revisions via updates | Function revisions; workflow updates | Function versions; durable orchestration code deployments |

### Category: Invocation semantics and lifecycle

| Capability | Hyperspot Serverless Runtime | AWS | Google Cloud | Azure |
|---|---|---|---|---|
| Sync invocation | Supported (`mode: sync`) | Lambda: `RequestResponse` | HTTP-triggered functions are synchronous by default | HTTP-triggered functions are synchronous by default |
| Async invocation | Supported (`mode: async` + poll status) | Lambda: `Event`; Step Functions: start execution then poll/described execution | Workflows: start execution then poll; Functions: async via events/pubsub | Durable: start orchestration then query status |
| Dry-run | Supported (`dry_run: true`) without durable record | Lambda: `DryRun` (permission check) | Generally via validation/testing tools rather than a single universal API | Generally via validation/testing tools rather than a single universal API |
| Durable execution record shape | Unified `ExecutionRecord` for start + status + debug | Different response shapes per service | Different response shapes per service | Different response shapes per service |
| Cancel | Supported (`:cancel`) | Step Functions: stop execution; Lambda: no “cancel running invocation” | Workflows: cancel execution; Functions: stop depends on trigger/runtime | Durable: terminate instance |
| Replay | Supported (`:replay`) creates new job from original params | Step Functions: redrive or re-execute manually | Workflows: re-execute manually | Durable: rewind or restart orchestration |
| Pause/resume | Supported (`:pause`, `:resume`) with `status: paused` for user-initiated pause or waiting on timers/external events | Step Functions: wait states + event-driven patterns; Lambda: event-driven via triggers | Workflows support waiting; functions are event-triggered or use separate services | Durable: timers + external events |
| Idempotency | Supported via idempotency key on start | Step Functions supports idempotency via execution name constraints; Lambda typically handled externally | Typically handled externally per trigger/service | Typically handled externally per trigger/service |

### Category: Observability and timeline

| Capability | Hyperspot Serverless Runtime | AWS | Google Cloud | Azure |
|---|---|---|---|---|
| Timeline / step history | Dedicated timeline endpoint (`/timeline`) | Step Functions execution history | Workflows execution history | Durable Functions history/events |
| Correlation and tracing | `observability.correlation_id` + `trace_id` fields | CloudWatch + X-Ray trace IDs (service-dependent) | Cloud Logging + Trace IDs | Application Insights correlation/tracing |
| Compute/memory metrics | Returned in `observability.metrics` (duration, billed duration, CPU time, memory limit/usage) | Lambda reports duration/billed duration/max memory used; CPU time not universally exposed | Execution metrics available via monitoring; CPU time not always directly exposed | Metrics via App Insights/monitoring; CPU time not always directly exposed |

### Category: Debugging and call trace

| Capability | Hyperspot Serverless Runtime | AWS | Google Cloud | Azure |
|---|---|---|---|---|
| Debug endpoint | `GET /executions/{id}/debug` returns execution record + `debug` section (`location`, `stack`) | No single universal debug endpoint; relies on logs, traces, and per-service UIs | No single universal debug endpoint; relies on logs/traces/UIs | No single universal debug endpoint; relies on logs/App Insights/Durable history |
| Execution trace | `GET /executions/{id}/trace` returns paginated ordered call list including params, duration, and exact response | Achievable via X-Ray subsegments/instrumentation; not a standard structured API output | Achievable via tracing/instrumentation; not a standard structured API output | Achievable via Durable history/instrumentation; not a standard structured API output |
| Completed-execution debug | Supported (`GET /debug` with `location`/`stack` null; `GET /trace` for full trace) | Historical logs/traces; UI varies by service | Historical logs/traces; UI varies by service | Historical logs/traces; durable history |

### Category: Error taxonomy

| Capability | Hyperspot Serverless Runtime | AWS | Google Cloud | Azure |
|---|---|---|---|---|
| Standard error envelope | Base error type `gts.x.core.faas.err.v1~` with `id`, `message`, `details` | Service-specific error payloads (varies) | Service-specific error payloads (varies) | Service-specific error payloads (varies) |
| Structured error classes | Derived errors: upstream HTTP, runtime limits/timeouts, code errors | Often string-based error+cause (Step Functions) and runtime-specific error types/statuses | Runtime-specific error types/statuses | Runtime-specific error types/statuses |

### Category: Caching and client behavior

| Capability | Hyperspot Serverless Runtime | AWS | Google Cloud | Azure |
|---|---|---|---|---|
| Result caching (TTL) | `traits.caching.max_age_seconds` defines client caching TTL for successful results | No first-class “function result cache TTL”; caching typically external (API gateway/CDN/app cache) | No first-class “function result cache TTL”; caching typically external | No first-class “function result cache TTL”; caching typically external |

### Open-source landscape (context)

Open-source systems that overlap with parts of this ADR:
- Temporal: strong durable workflow orchestration, rich history and retries; different model (SDK-first, not GTS-schema-first) and not a unified “function+workflow definition schema”.
- Apache Airflow: batch/workflow scheduling with strong DAG semantics; less aligned with request/response serverless invocation and per-invocation records.
- Knative / OpenFaaS: function execution and autoscaling; workflows/orchestration typically handled by separate systems.
