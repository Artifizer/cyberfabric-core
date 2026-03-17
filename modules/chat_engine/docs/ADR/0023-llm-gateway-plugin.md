Created:  2026-03-06 by Constructor Tech
Updated:  2026-03-09 by Constructor Tech
---
status: accepted
date: 2026-03-06
---

# ADR-0023: LLM Gateway Plugin


<!-- toc -->

- [Context and Problem Statement](#context-and-problem-statement)
- [Decision Drivers](#decision-drivers)
- [Considered Options](#considered-options)
- [Decision Outcome](#decision-outcome)
  - [Plugin Lifecycle](#plugin-lifecycle)
  - [External Service Dependencies](#external-service-dependencies)
  - [Consequences](#consequences)
  - [Confirmation](#confirmation)
- [Pros and Cons of the Options](#pros-and-cons-of-the-options)
  - [Option 1: Model Registry + GTS derived schemas (chosen)](#option-1-model-registry--gts-derived-schemas-chosen)
  - [Option 2: Hardcoded capabilities + GTS derived schemas](#option-2-hardcoded-capabilities--gts-derived-schemas)
  - [Option 3: All config in SessionType.metadata](#option-3-all-config-in-sessiontypemetadata)
- [Capability Resolution via Model Registry](#capability-resolution-via-model-registry)
  - [Capability Refresh on Model Change (`on_session_updated`)](#capability-refresh-on-model-change-onsessionupdated)
- [Schema Extensions](#schema-extensions)
  - [Metadata Schemas](#metadata-schemas)
  - [Entity Schemas](#entity-schemas)
- [Traceability](#traceability)

<!-- /toc -->

**ID**: `cpt-cf-chat-engine-adr-llm-gateway-plugin`

## Context and Problem Statement

Chat Engine defines a generic plugin interface (`ChatEngineBackendPlugin` trait, ADR-0022) for backend integrations. The first concrete plugin is the **LLM gateway plugin** — it connects Chat Engine to an LLM gateway service and a Model Registry service. The plugin must solve three concerns without modifying Chat Engine core:

1. **Capability resolution** — determine which LLM parameters (model, temperature, max_tokens, web_search) are available for a given session type and expose them through the capabilities system (ADR-0002)
2. **Schema extension** — store LLM-specific data (response facts, token usage, plugin configuration) in Chat Engine's `metadata` JSONB fields with typed validation
3. **Message processing** — forward user messages to the LLM gateway service and stream responses back

How should the LLM gateway plugin implement these concerns while keeping Chat Engine agnostic to LLM specifics?

## Decision Drivers

* Capabilities must come from a reliable external source — hardcoding them in the plugin creates drift when models change
* User-selectable LLM params (model, temperature, max_tokens, web_search) must go through the capabilities system (ADR-0002) — plugin resolves `Session.enabled_capabilities` at session creation by querying Model Registry
* Plugin configuration belongs in `SessionType.metadata` — opaque to Chat Engine; model selection is not part of session type config — the default model is determined by Model Registry
* LLM response facts (model_used, finish_reason, temperature_used) belong in `Message.metadata`
* Base `Usage` schema must remain abstract and unchanged — `LlmUsage` is a standalone schema nested inside `LlmMessageMetadata.usage` as a plain dict within the JSONB field, not a derived type of `Usage`
* Schema validation must work without modifying Chat Engine core
* LLM plugin schema namespace must be isolated from other plugins

## Considered Options

* **Option 1: Model Registry + GTS derived schemas** — capabilities fetched from Model Registry at configuration time; LLM-specific metadata via registered GTS derived types; message processing via LLM gateway HTTP calls
* **Option 2: Hardcoded capabilities + GTS derived schemas** — capabilities defined as constants in plugin code; same schema extension approach
* **Option 3: All config in SessionType.metadata** — no capabilities for LLM params; everything in developer config; user cannot override per-session; flat untyped metadata

## Decision Outcome

Chosen option: "Model Registry + GTS derived schemas", because it keeps capabilities in sync with actual model support, separates user-selectable concerns (capabilities) from developer configuration (SessionType.metadata), provides typed validation for LLM-specific fields, and keeps the LLM plugin namespace isolated.

### Plugin Lifecycle

1. **Startup** — plugin registers GTS derived schemas (`LlmSessionTypeMetadata`, `LlmMessageMetadata`, `LlmUsage`) and entity schemas in the GTS schema registry
2. **Session type configuration** (`on_session_type_configured`) — plugin validates `SessionType.metadata` and returns an empty `Vec<Capability>` — capability resolution is deferred to session creation
3. **Session creation** (`on_session_created`) — plugin performs two-step capability resolution via **Model Registry**:
   1. Queries Model Registry for the **list of available models** — the registry returns the models list along with the designated default model; builds a `model` capability (`type: "enum"`, `enum_values` from registry, `default_value` from registry's default)
   2. Queries Model Registry for **capabilities of the default model** (temperature, max_tokens, web_search, etc.) and builds additional capabilities from the response
   3. Returns the combined `Vec<Capability>` — Chat Engine stores them as `Session.enabled_capabilities`
4. **Session capability update** (`on_session_updated`) — when the user selects a different model via the capabilities UI, Chat Engine calls the plugin with the updated `CapabilityValue[]`. The plugin:
   1. Detects that the `model` capability value has changed
   2. Queries Model Registry for **capabilities of the newly selected model**
   3. Returns updated `Vec<Capability>` — the `model` capability preserves its `enum_values` (available models list unchanged), but model-specific capabilities (temperature, max_tokens, web_search, etc.) are replaced with the new model's defaults and constraints
   4. Chat Engine overwrites `Session.enabled_capabilities` with the returned set
5. **Message processing** (`on_message`, `on_message_recreate`) — plugin builds an LLM gateway request from the message context and user-selected `CapabilityValue[]`, calls the LLM gateway service via HTTP, and streams the response back as `ResponseStream`
6. **Summarization** (`on_session_summary`) — plugin routes summary requests to the LLM gateway service

### External Service Dependencies

| Service | Used In | Purpose |
|---------|---------|---------|
| **Model Registry** | `on_session_created`, `on_session_updated` | `on_session_created`: Step 1 — retrieve list of available models; Step 2 — retrieve capabilities for the default model. `on_session_updated`: retrieve capabilities for the newly selected model |
| **LLM Gateway** | `on_message`, `on_message_recreate`, `on_session_summary` | Forward messages and receive streamed LLM responses |

### Consequences

* Good, because capabilities reflect actual model support — Model Registry is the single source of truth
* Good, because adding a new model or changing model parameters requires no plugin code changes
* Good, because users can select model, temperature, max_tokens, and web_search per session via the capabilities UI
* Good, because capabilities are resolved at session creation time — each session gets a fresh model list and model-specific capabilities from Model Registry
* Good, because `LlmUsage` provides typed token counts (prompt/completion/cached) without breaking the abstract base `Usage` schema
* Good, because Chat Engine validates LLM metadata blobs against registered GTS schemas (FR-021)
* Good, because plugin schema namespace is isolated (`gts.x.chat_engine.llm_gateway.*`) — no conflicts with other plugins
* Good, because base schemas remain unchanged — non-LLM plugins are unaffected
* Bad, because plugin depends on Model Registry availability during `on_session_created` — session creation fails if Model Registry is down
* Bad, because plugin must register GTS schemas at startup before any session type can be created
* Bad, because Chat Engine must implement schema registry lookup for metadata validation (FR-021 is `p2` — not yet implemented)

### Confirmation

Confirmed when:

- LLM plugin registers `LlmSessionTypeMetadata`, `LlmMessageMetadata`, and `LlmUsage` in GTS at startup
- LLM plugin queries Model Registry during `on_session_created` (two-step: available models list, then default model capabilities) and returns session-specific capabilities
- LLM plugin queries Model Registry during `on_session_updated` when the user changes the `model` capability, and returns updated capabilities for the new model
- Creating a session type with LLM plugin validates `SessionType.metadata` against `LlmSessionTypeMetadata` (currently empty schema)
- Assistant message responses include `Message.metadata` with `model_used`, `finish_reason`, and `LlmUsage` token counts
- Non-LLM session types are unaffected by LLM schema registration
- `on_message` successfully calls LLM gateway and streams response back through Chat Engine

## Pros and Cons of the Options

### Option 1: Model Registry + GTS derived schemas (chosen)

Capabilities from Model Registry; typed metadata via GTS derived types; LLM gateway HTTP calls for message processing.

* Good, because capabilities stay in sync with model support automatically
* Good, because user control over LLM params per session via standard capabilities UI
* Good, because schema validation without Chat Engine core changes
* Good, because plugin namespace isolation prevents schema conflicts
* Bad, because Model Registry must be available during session creation
* Bad, because requires FR-021 (schema-extensibility) implementation before metadata validation is active

### Option 2: Hardcoded capabilities + GTS derived schemas

Capabilities defined as constants in plugin code; same schema extension approach.

* Good, because no external dependency for capability resolution
* Good, because schema validation same as Option 1
* Bad, because capability definitions drift when models are added or changed
* Bad, because plugin code changes required for every model update
* Bad, because different deployments cannot have different model catalogs without code forks

### Option 3: All config in SessionType.metadata

LLM params all in developer config; no capabilities; flat untyped metadata.

* Good, because simpler — no capability declarations, no schema registration
* Bad, because users cannot change model or temperature per session
* Bad, because no validation — typos and type mismatches silently accepted
* Bad, because no namespace isolation — metadata conflicts between plugins possible

## Capability Resolution via Model Registry

During `on_session_created`, the LLM gateway plugin performs two-step capability resolution:

**Step 1 — Available Models List**:
1. Queries the **Model Registry** service for the list of all available models — the response includes the models list and the designated default model
2. Builds a `model` capability: `{ id: "model", type: "enum", enum_values: [models from registry], default_value: [default from registry] }`
3. Stores the `model` capability in the session's `enabled_capabilities`

**Step 2 — Default Model Capabilities**:
1. Queries the **Model Registry** service for capabilities of the default model (from Step 1)
2. Model Registry returns model-specific parameters (temperature, max_tokens, web_search, etc.) with allowed values and defaults
3. Maps the response to additional `Capability` entries
4. Appends them to the session's `enabled_capabilities`

### Capability Refresh on Model Change (`on_session_updated`)

When the user selects a different model in the UI (updates the `model` capability value), Chat Engine calls `plugin.on_session_updated(ctx)` with the updated `CapabilityValue[]`. The LLM gateway plugin:

1. Compares the new `model` value with the current `model` default in `Session.enabled_capabilities`
2. If changed — queries the **Model Registry** for capabilities of the newly selected model
3. Rebuilds capabilities: keeps the `model` capability with its `enum_values` intact (available models list is unchanged), updates `default_value` to the new model, and replaces model-specific capabilities (temperature, max_tokens, web_search, etc.) with the new model's parameters
4. Returns `Vec<Capability>` — Chat Engine overwrites `Session.enabled_capabilities`

If the `model` value did not change, the plugin returns the existing capabilities unchanged.

---

The actual set of capabilities and their `enum_values` / defaults depend on the model's entry in the Model Registry — different models may expose different capabilities.

Example result after both steps for a typical LLM model:

- `{ id: "model", name: "AI Model", type: "enum", default_value: "gpt-4o", enum_values: ["gpt-4o", "gpt-4o-mini", "o1"] }` — from Step 1
- `{ id: "temperature", name: "Temperature", type: "int", default_value: 70 }` — from Step 2, integer 0–100 maps to 0.0–1.0
- `{ id: "max_tokens", name: "Max Tokens", type: "int", default_value: 4096 }` — from Step 2
- `{ id: "web_search", name: "Web Search", type: "bool", default_value: false }` — from Step 2

## Schema Extensions

### Metadata Schemas

**GTS Schema IDs registered by LLM gateway plugin**:

| Schema | GTS ID | Extension Point |
|--------|--------|-----------------|
| `LlmSessionTypeMetadata` | `gts://gts.x.chat_engine.llm_gateway.session_type_metadata.v1` | `SessionType.metadata` |
| `LlmMessageMetadata` | `gts://gts.x.chat_engine.llm_gateway.message_metadata.v1` | `Message.metadata` |
| `LlmUsage` | `gts://gts.x.chat_engine.llm_gateway.usage.v1` | nested in `LlmMessageMetadata.usage` |

**`LlmSessionTypeMetadata` fields**: (no fields currently — reserved for future plugin configuration)

**`LlmMessageMetadata` fields**: `model_used: string`, `finish_reason: enum[stop|length|content_filter|tool_calls]`, `temperature_used?: number`, `usage?: LlmUsage`

**`LlmUsage` fields**: `prompt_tokens: int`, `completion_tokens: int`, `total_tokens: int`, `cached_tokens?: int`

### Entity Schemas

GTS entity schemas registered by LLM gateway plugin (extend base Chat Engine schemas via JSON Schema `allOf`, overriding the `metadata` property; `metadata` is stored as JSONB):

| Schema | GTS ID | Extends |
|--------|--------|---------|
| `LlmMessage` | `gts://gts.x.chat_engine.llm_gateway.message.v1` | `common/Message` |
| `LlmSessionType` | `gts://gts.x.chat_engine.llm_gateway.session_type.v1` | `common/SessionType` |
| `LlmMessageGetResponse` | `gts://gts.x.chat_engine.llm_gateway.message_get_response.v1` | `message/MessageGetResponse` |
| `LlmMessageNewResponse` | `gts://gts.x.chat_engine.llm_gateway.message_new_response.v1` | `webhook/MessageNewResponse` |
| `LlmMessageRecreateResponse` | `gts://gts.x.chat_engine.llm_gateway.message_recreate_response.v1` | `webhook/MessageRecreateResponse` |
| `LlmStreamingCompleteEvent` | `gts://gts.x.chat_engine.llm_gateway.streaming_complete_event.v1` | `streaming/StreamingCompleteEvent` |

## Traceability

- **PRD**: [PRD.md](../PRD.md)
- **DESIGN**: [DESIGN.md](../DESIGN.md)

* `cpt-cf-chat-engine-fr-schema-extensibility` — GTS derived schema registration is the mechanism used to extend metadata fields
* `cpt-cf-chat-engine-adr-plugin-backend-integration` — plugin system and trait interface (ADR-0022)
* `cpt-cf-chat-engine-adr-capability-model` — capabilities for user-selectable LLM params (ADR-0002)
* `cpt-cf-chat-engine-adr-session-metadata` — JSONB extension point and GTS validation strategy (ADR-0017)