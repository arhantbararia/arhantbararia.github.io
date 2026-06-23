---
layout: post
title: "Sequels.diy compatible Action development guide"
date: 2026-06-22
categories: sequels action dev-guide
---
This guide provides a concise, language-agnostic specification and implementation blueprint for building Sequels.diy-compatible action plugins. It covers architecture, data models, HTTP endpoints, RabbitMQ consumption, task execution, and deployment patterns so you can implement reliable actions in any language that supports HTTP and RabbitMQ AMQP.

> **Language-agnostic specification.** All examples use JSON for data structures. Implement in any language that can serve HTTP, connect to RabbitMQ (AMQP 0-9-1), and make outbound HTTP requests.

---

## Table of Contents

1. [Pre-Development Checklist](#1-pre-development-checklist)
2. [Architecture Overview](#2-architecture-overview)
3. [Trigger vs. Action: Key Differences](#3-trigger-vs-action-key-differences)
4. [Directory Structure](#4-directory-structure)
5. [File-by-File Specification](#5-file-by-file-specification)
6. [Data Models](#6-data-models)
7. [Workflow Executor Registration](#7-workflow-executor-registration)
8. [HTTP Endpoints](#8-http-endpoints)
9. [RabbitMQ Consumption (Receiving Tasks)](#9-rabbitmq-consumption-receiving-tasks)
10. [Template Resolution](#10-template-resolution)
11. [Task Execution & Capability Routing](#11-task-execution--capability-routing)
12. [Result Publishing](#12-result-publishing)
13. [Retry & Error Handling](#13-retry--error-handling)
14. [Authentication & Token Refresh](#14-authentication--token-refresh)
15. [Environment Variables](#15-environment-variables)
16. [Deployment](#16-deployment)
17. [End-to-End Walkthrough](#17-end-to-end-walkthrough)
18. [Quick Reference Tables](#18-quick-reference-tables)
19. [Design Principles](#19-design-principles)

---

## 1. Pre-Development Checklist

Before writing any code, confirm every item below:

- [ ] **Plugin name** decided — format: `<service>_action` (e.g. `slack_action`, `spotify_action`)
- [ ] **Route prefix** decided — format: `/<service>/action` (e.g. `/slack/action`)
- [ ] **Capabilities list** finalized — each capability needs:
  - [ ] `unique_key` — globally unique string (e.g. `slack_post_message`, `spotify_add_to_queue`)
  - [ ] `name` — human-readable label
  - [ ] `description` — one-sentence explanation
  - [ ] `config_schema` — map of parameter names → types the user must supply
  - [ ] `output_schema` — map of output field names → types returned after execution
- [ ] **Auth type** decided — one of: `"None"`, `"OAuth 2.0"`, `"Personal Access Token"`, `"API Key"`, `"Bot Token"`
- [ ] **Token refresh** strategy decided — does the service use expiring OAuth2 tokens that need refresh, or non-expiring tokens (bot tokens, API keys)?
- [ ] **Retry strategy** decided — retry count, delay between retries, which HTTP status codes trigger a retry
- [ ] **RabbitMQ access** confirmed — connection string available
- [ ] **Workflow executor URL** known — default `http://localhost:8082`
- [ ] **Directory structure** created (see §4)
- [ ] **Dockerfile** written (standalone) or entry added to unified `Dockerfile` + `active_plugins.txt`
- [ ] **Environment variables** documented

---

## 2. Architecture Overview

```
                        ┌──────────────────────────┐
                        │    workflow_executor      │
                        │  (orchestration backend)  │
                        └──────┬───────────▲────────┘
                    POST /register     POST /<svc>/action/setup
                    (on startup)       POST /<svc>/action/remove
                               │               │
                        ┌──────▼───────────────┐
                        │   YOUR ACTION PLUGIN  │
                        │  HTTP server on :PORT  │
                        └──────────┬────────────┘
                                   │ consume from RabbitMQ queue
                                   │ (1 consumer per action instance)
                                   │
                        ┌──────────▼──────────┐
                        │      RabbitMQ        │
                        │ Queue: workflow_<id>_action │
                        │ (bound to EVENT_MESSAGE     │
                        │  exchange, key=workflow_id)  │
                        └──────────┬──────────┘
                                   │ task received
                                   ▼
                        ┌──────────────────────┐
                        │   External Service   │
                        │  (Slack, Spotify…)     │
                        └──────────┬───────────┘
                                   │ API call made
                                   ▼
                        ┌──────────────────────┐
                        │      RabbitMQ         │
                        │ Exchange: ACTION_RESULT│
                        │ Routing key: workflow_id│
                        └───────────────────────┘
```

**Lifecycle:**
1. Plugin starts → registers with `workflow_executor` via `POST /register`
2. Executor calls `POST /<svc>/action/setup` with config + queue name → plugin starts a RabbitMQ consumer on the assigned queue
3. Trigger fires → executor publishes `ActionTask` to the queue → consumer receives it
4. Consumer resolves `{{trigger.payload.<key>}}` templates in config, executes the action against the external service
5. Consumer publishes `ActionResult` to the `ACTION_RESULT` exchange with routing key = `workflow_id`
6. Consumer `Ack`s the message (success) or `Nack`s (permanent failure after retries)
7. Executor calls `POST /<svc>/action/remove` → plugin stops the consumer and cleans up

---

## 3. Trigger vs. Action: Key Differences

Understanding how actions differ from triggers is essential before you begin.

| Aspect | Trigger Plugin | Action Plugin |
|---|---|---|
| **Direction** | Detects external events → pushes data into system | Receives tasks from system → calls external APIs |
| **RabbitMQ role** | **Publisher** — publishes `TriggerEvent` to `EVENT_MESSAGE` | **Consumer** — consumes `ActionTask` from a dedicated queue; **Publisher** — publishes `ActionResult` to `ACTION_RESULT` |
| **Worker type** | Poller/Scheduler (goroutine with ticker) | Consumer (blocking loop reading from queue) |
| **`container_type`** | `"trigger"` | `"action"` |
| **Queue name** | Not used (publishes to exchange directly) | Assigned by executor: `workflow_<id>_action` |
| **Template resolution** | N/A | Must resolve `{{trigger.payload.X}}` tokens in config before executing |
| **Result reporting** | N/A (trigger fires and forgets) | Must publish `ActionResult` for success, error, and retry status |
| **Deduplication** | Yes (seen-set or timestamp) | No (each message is processed exactly once) |

---

## 4. Directory Structure

```
<service>_action/
├── main.go                    # HTTP server, route registration, global state, handlers
├── go.mod                     # Module definition (or equivalent for your language)
├── go.sum                     # Dependency lock file
├── Dockerfile                 # Standalone container build (optional)
├── models/
│   └── models.go              # All data structures: ActionConfig, AuthData, ActionTask,
│                              #   ActionResult, RegistrationRequest, PluginCapability
├── services/
│   ├── registration.go        # Self-registration logic (POST to workflow_executor /register)
│   └── <service>.go           # Business logic: service client, capability implementations,
│                              #   template resolution, task router, auth helpers
└── worker/                    # (or workers/)
    ├── consumer.go            # RabbitMQ consumer: connect, consume loop, reconnection,
    │                          #   graceful shutdown
    └── publisher.go           # RabbitMQ publisher for ActionResult messages
```

---

## 5. File-by-File Specification

### 5.1 `main.go` — Entry Point & HTTP Server

**Responsibilities:**
- Declare global state: service instance, result publisher, consumer map, config store (thread-safe), mutex
- Define `SetupPayload` and `RemovePayload` structs for JSON decoding
- Implement `buildActionConfig()` to extract typed config from raw JSON map (including `_auth_context` extraction and `RawConfig` preservation for template resolution)
- Implement `workflowConfigProvider` — a struct that satisfies the `ConfigProvider` interface by reading from/writing to the global config store
- Implement HTTP handlers: `handleSetup`, `handleRemove`, `handleValidate`, `handleHealth`
- On startup: initialize service, start background registration with retry, register HTTP routes, listen on port

**Global state pattern:**
```
serviceSvc      — singleton service client (HTTP client with proxy support)
resultPublisher — singleton RabbitMQ publisher for ActionResults
consumers       — map[string]*Consumer   (keyed by action instance ID)
configs         — thread-safe map        (keyed by action instance ID)
mu              — mutex for consumers map
consumerCount   — atomic counter for logging
```

**`workflowConfigProvider` interface implementation:**
```
interface ConfigProvider:
    GetConfig(id: string) -> (ActionConfig, error)
        — Load config from the global thread-safe map by ID
    UpdateAuth(id: string, auth: map[string]AuthData) -> error
        — Update the auth context in the stored config (for token refresh)
```

This provider is passed to the service's task router so that the consumer goroutine can read/update config without direct access to global state.

**`buildActionConfig(payload)` logic:**
```
1. Set capability_key from payload
2. Extract _auth_context:
   a. Try direct type assertion to map[string]AuthData
   b. Fallback: re-marshal + unmarshal the raw _auth_context value
3. Build RawConfig: copy all config keys EXCEPT _auth_context
4. Extract service-specific typed fields from the raw config map
   (e.g. "channel" → cfg.Channel, "message" → cfg.Message)
5. Return the populated ActionConfig
```

**Port resolution order:** `PLUGIN_LISTEN_PORT` → `PLUGIN_PORT` → `"8080"` (default)

**Registration retry:** Up to 10 attempts with linear backoff (`(attempt) * 5 seconds`), run in a background thread so the HTTP server starts immediately.

---

### 5.2 `models/models.go` — Data Structures

Defines all shared types. See §6 for full schemas.

---

### 5.3 `services/registration.go` — Executor Registration

**Responsibilities:**
- Build a `RegistrationRequest` with plugin metadata and capabilities list
- POST it as JSON to `{EXECUTOR_URL}/register`
- Build a unique plugin ID: `{HOSTNAME}-<service>-action` (HOSTNAME is set by Docker; fallback to `{host}:{port}-<service>-action`)

---

### 5.4 `services/<service>.go` — Business Logic & Task Router

This is the core of the action plugin. It contains:

**Service struct:**
- Holds an HTTP client (with proxy support, timeouts, optional IPv4-only dialing)
- Configurable retry count and retry delay (from env vars `RETRY_COUNT`, `RETRY_SECONDS`)

**Interfaces defined (for testability):**
```
interface ConfigProvider:
    GetConfig(id) -> (ActionConfig, error)
    UpdateAuth(id, auth) -> error

interface PublisherProvider:
    Publish(workflowID, ActionResult) -> error
```

**Template resolution function (`resolveTemplates`):**
- Matches all `{{trigger.payload.<key>}}` patterns in string config values
- Replaces them with actual values from the trigger event payload
- Deep-copies `RawConfig` before mutation to avoid corrupting stored templates
- Re-extracts typed fields from resolved raw config
- See §10 for detailed specification

**Auth helper (`GetValidAuth`):**
- Extracts credentials from the auth context
- For OAuth2 services: checks token expiry and refreshes if needed (see §14)
- For non-expiring tokens (Bot Tokens, API Keys): returns directly

**Result publisher helper (`publishResult`):**
- Builds an `ActionResult` from task metadata, output, error, and status
- Publishes to RabbitMQ via the `PublisherProvider` interface
- See §12 for detailed specification

**Task router (`HandleTaskRouter`):**
- Returns a closure (callback function) that the RabbitMQ consumer calls for each message
- The closure captures a copy of the initial `ActionConfig` (assigned at setup) — this ensures independent state per consumer instance
- On each message:
  1. Unmarshal the message body into an `ActionTask`
  2. Deep-copy the config, then resolve templates with the task's trigger payload
  3. Get valid auth credentials (refreshing if needed)
  4. Route to the correct capability function based on `capability_key`
  5. Execute with retry loop (see §13)
  6. Publish result (`success`, `retrying`, or `error`)
  7. Ack on success, Nack on permanent failure

**Per-capability functions:**
- Each function takes auth credentials + resolved config, calls the external API, returns `(output_map, elapsed_ms, error)`
- The output map keys MUST match the `output_schema` declared during registration

---

### 5.5 `worker/consumer.go` — RabbitMQ Consumer

**Responsibilities:**
- Connect to RabbitMQ using URL parameter
- Declare the queue (ensure it exists)
- Start consuming messages with a consumer tag
- Pass each delivered message to the `TaskHandler` callback
- Handle reconnection on connection loss (retry with 5-second delay)
- Graceful shutdown via stop channel/signal

**Key types:**
```
type TaskHandler = func(delivery)
    — The callback function. Implementations MUST Ack or Nack the delivery.

type Consumer:
    url         — RabbitMQ connection string
    queueName   — the assigned queue to consume from
    consumerTag — unique tag for this consumer (e.g. "slack-action-<workflow_id>")
    handler     — TaskHandler callback
    conn        — RabbitMQ connection (protected by mutex)
    done        — channel signaling shutdown
    wg          — WaitGroup for graceful exit
```

**`Start()`** — launches the consumer loop in a background goroutine

**`Stop()`** — closes the done channel (with once-guard to prevent double-close), closes the connection (which unblocks the delivery loop), then waits for the goroutine to exit

**Reconnection loop pattern:**
```
run():
    loop:
        if done is signaled: return
        err = connectAndConsume()
        if err: log warning
        wait 5 seconds or until done is signaled

connectAndConsume():
    connect to RabbitMQ
    open channel
    declare queue (durable, not auto-delete)
    start consuming (manual ack, not exclusive)
    for each delivery:
        call handler(delivery)
    — loop exits when connection closes
    if done was signaled: return nil (graceful)
    else: return error (unexpected disconnect)
```

---

### 5.6 `worker/publisher.go` — ActionResult Publisher

**Responsibilities:**
- Connect to RabbitMQ using `RABBITMQ_URL` env var (default: `amqp://guest:guest@localhost:5672/`)
- Maintain connection + channel with lazy reconnection (`ensureConnected()`)
- `Publish(workflowID, result)` — JSON-marshal the `ActionResult`, publish to exchange `"ACTION_RESULT"` with routing key = `workflowID`, content type `application/json`, delivery mode persistent
- Thread-safe via mutex
- On publish failure: invalidate connection so the next attempt reconnects

---

## 6. Data Models

### 6.1 `ActionConfig`

Holds all configuration for one action instance.

```json
{
  "capability_key": "string",
  "_auth_context": {
    "<provider_name>": { AuthData }
  },
  "raw_config": {
    // All config fields EXCEPT _auth_context, preserved for template resolution.
    // Values may contain {{trigger.payload.X}} templates.
  }
  // ...service-specific typed fields (see below)
}
```

**Service-specific fields examples:**
| Service | Fields |
|---|---|
| Slack | `channel` (string), `message` (string), `attachments` (string), `link` (string) |
| Telegram | `chat_id` (string), `message` (string), `text` (string), `caption` (string), `file_url` (string) |
| Spotify | `track_id` (string), `track_query` (string), `playlist_id` (string) |
| X (Twitter) | `tweet_text` (string), `image_url` (string) |
| Google Sheets | `spreadsheet_id` (string), `sheet_name` (string), `data` (string) |
| WhatsApp | `phone_number` (string), `message` (string) |

### 6.2 `AuthData`

Universal credential envelope. Plugins use whichever fields apply.

```json
{
  "access_token": "string",
  "refresh_token": "string",
  "token_type": "string",
  "expiry": "ISO8601 datetime",
  "provider": "string",
  "external_account_id": "string",
  "api_key": "string",
  "bot_token": "string",
  "secret_key": "string",
  "username": "string",
  "password": "string"
}
```

### 6.3 `ActionTask`

Consumed from RabbitMQ when a trigger fires. This is the message your consumer will receive on the queue.

```json
{
  "id": "uuid-v4",
  "workflow_id": "string",
  "trigger_id": "string",
  "type": "event",
  "name": "Human-Readable Trigger Name",
  "capability_key": "string",
  "payload": {
    // keys from the trigger's output_schema
    // e.g. { "title": "...", "url": "...", "score": "..." }
  },
  "timestamp": "ISO8601 datetime (UTC)"
}
```

**Critical:** This is the same `TriggerEvent` published by trigger plugins. The `payload` contains the trigger's output, which your action uses for template resolution. The `capability_key` in this message refers to the **trigger's** capability, not the action's. Your action's capability key comes from the stored `ActionConfig`.

### 6.4 `ActionResult`

Published to RabbitMQ after action execution to allow the executor to track job outcomes.

```json
{
  "task_id": "string (matches ActionTask.id)",
  "workflow_id": "string",
  "success": true,
  "status": "success | error | retrying",
  "output": {
    // keys must match the output_schema declared at registration
    // only populated on success
  },
  "error": "error message string (only on error/retrying)",
  "retry_count": 0,
  "response_time_ms": 1234,
  "timestamp": "ISO8601 datetime (UTC)",
  "metadata": {
    // optional additional metadata
  }
}
```

**Status values:**
| Status | Meaning | `success` field | `output` field | `error` field |
|---|---|---|---|---|
| `"success"` | Action completed successfully | `true` | Populated | Empty |
| `"retrying"` | Attempt failed, will retry | `false` | Empty | Populated |
| `"error"` | All retries exhausted, permanent failure | `false` | Empty | Populated |

### 6.5 `RegistrationRequest`

Sent to `workflow_executor` at startup.

```json
{
  "id": "unique-plugin-instance-id",
  "name": "Human-Readable Plugin Name",
  "container_type": "action",
  "plugin_provider_service": "ServiceName",
  "plugin_host": "hostname-or-ip",
  "plugin_port": "port-string",
  "endpoints": {
    "setup":    "/<service>/action/setup",
    "remove":   "/<service>/action/remove",
    "health":   "/<service>/action/health"
  },
  "auth_types": ["None" | "OAuth 2.0" | "OAUTH2" | "API Key" | ...],
  "capabilities": [ PluginCapability, ... ]
}
```

**Important:** `container_type` MUST be `"action"` (not `"trigger"`).

### 6.6 `PluginCapability`

```json
{
  "unique_key": "service_capability_name",
  "name": "Human-Readable Name",
  "description": "One-sentence description.",
  "component_type": "ACTION",
  "config_schema": {
    "param_name": "type_string"
  },
  "output_schema": {
    "output_field": "type_string"
  }
}
```

- `component_type` MUST be `"ACTION"` (not `"TRIGGER"`).
- `config_schema`: keys = config parameter names the user supplies. Values = type hints (`"string"`, `"integer"`, `"boolean"`). The schema can use either simple type strings or rich objects with `type`, `required`, and `description` fields.
  - **Simple format**: `{"param_name": "string"}`
  - **Rich format**: `{"param_name": {"type": "string", "required": true, "description": "Help text"}}`
- `output_schema`: keys = output field names returned after action execution. These are stored for job tracking and downstream use.

---

## 7. Workflow Executor Registration

### 7.1 When

Immediately after the HTTP server is ready to accept requests. Run in a background thread so the server isn't blocked.

### 7.2 Endpoint

```
POST {EXECUTOR_URL}/register
Content-Type: application/json
Body: RegistrationRequest (see §6.5)
```

### 7.3 Response

- **2xx** — success, plugin is registered. The executor will:
  1. Upsert a `TaskContainer` record in its database
  2. Sync the plugin's capabilities to the `goat_backend` (so the frontend can discover available actions)
- **Non-2xx** — retry with backoff

### 7.4 Retry Strategy

```
for attempt in 1..10:
    response = POST /register
    if response.status is 2xx:
        return success
    wait (attempt * 5) seconds
log warning: "Could not register after 10 attempts"
```

### 7.5 Plugin ID Construction

Since all plugins may run inside a single container sharing the same `HOSTNAME` env var, each plugin MUST append a unique suffix:

```
id = HOSTNAME + "-<service>-action"    // e.g. "abc123-slack-action"
```

Fallback when HOSTNAME is empty:
```
id = PLUGIN_HOST + ":" + PLUGIN_PORT + "-<service>-action"
```

### 7.6 What Happens After Registration (Executor Side)

Once registered, the executor:
1. Stores the plugin as a `TaskContainer` with `container_type: "action"` in its database
2. Syncs to `goat_backend` via `POST /api/v1/internal/plugins/sync` with:
   - `provider_name`, `host`, `port`, `auth_types`, `capabilities`
3. When a user creates a workflow, the executor:
   - Finds a healthy action container matching the requested `plugin_provider_service`
   - Uses least-loaded selection when multiple instances exist
   - Creates a dedicated RabbitMQ queue `workflow_<id>_action`
   - Binds the queue to `EVENT_MESSAGE` exchange with routing key = `workflow_id`
   - Calls `/setup` on the selected container
4. The executor also sets up monitoring queues and exchanges (`TRIGGER_RESULT`, `ACTION_RESULT`)

---

## 8. HTTP Endpoints

All endpoints use the prefix `/<service>/action` (e.g. `/slack/action`).

### 8.1 `POST /<service>/action/setup`

Called by `workflow_executor` to activate an action instance.

**Request Body:**
```json
{
  "id": "action-instance-uuid",
  "workflow_id": "workflow-uuid",
  "queue_name": "workflow_<workflow-id>_action",
  "capability_key": "service_capability_name",
  "config": {
    "param1": "value1 or {{trigger.payload.field}}",
    "param2": "value2",
    "_auth_context": {
      "provider-name": {
        "access_token": "...",
        "refresh_token": "...",
        "token_type": "Bearer",
        "expiry": "2026-06-01T00:00:00Z",
        "provider": "provider-name",
        "external_account_id": "...",
        "api_key": "...",
        "bot_token": "...",
        "secret_key": "...",
        "username": "...",
        "password": "..."
      }
    }
  }
}
```

**Notes:**
- `queue_name` is **required** for action plugins (unlike triggers). This is the RabbitMQ queue the consumer must subscribe to. The executor creates this queue before calling setup.
- `workflow_id` is **required**.
- `_auth_context` is injected by `workflow_executor` with decrypted user credentials. It is a map keyed by provider name.
- Config values may contain `{{trigger.payload.<key>}}` template tokens that are resolved at runtime when a task arrives.
- If a consumer already exists for this `id`, stop it before creating a new one (idempotent re-setup).

**Handler logic (step by step):**
```
1. Decode JSON payload
2. Validate: queue_name and workflow_id must be non-empty
3. Build ActionConfig from payload (buildActionConfig):
   a. Extract capability_key
   b. Extract _auth_context
   c. Build RawConfig (all keys except _auth_context)
   d. Extract service-specific typed fields
4. Store config in global map, keyed by action ID
5. Lock the consumers mutex
6. If consumer for this ID already exists, Stop() it
7. Create ConfigProvider and get/initialize the service + publisher singletons
8. Create TaskHandler via service.HandleTaskRouter(provider, publisher, seq, instanceID, config)
9. Create new Consumer(rabbitmqURL, queueName, consumerTag, taskHandler)
10. Start the consumer
11. Store consumer in global map
12. Return 200 OK with setup_complete response
```

**Response (200 OK):**
```json
{
  "success": true,
  "message": "setup_complete",
  "queue_name": "workflow_<id>_action",
  "seq": 1
}
```

**Error (400 Bad Request):** returned if JSON is malformed or required fields are missing.

---

### 8.2 `POST /<service>/action/remove`

Called by `workflow_executor` to deactivate an action instance.

**Request Body:**
```json
{
  "id": "action-instance-uuid",
  "workflow_id": "workflow-uuid"
}
```

**Removal logic (in order):**
1. If `id` is provided, look up and stop that specific consumer.
2. If `workflow_id` is provided, iterate all consumers and stop any matching that workflow (bulk removal fallback).
3. Clean up both the consumer map and the config store.

**Response (200 OK):**
```json
{
  "success": true,
  "message": "removed",
  "removed_count": 1
}
```

---

### 8.3 `POST /<service>/action/validate` *(optional but recommended)*

Called to check if an identical action is already running (duplicate detection).

**Request Body:** Same as `/setup`.

**Response (200 OK):**
```json
{
  "is_duplicate": true
}
```

**Logic:** Build an `ActionConfig` from the payload, compare it field-by-field (or via deep-equal) with the stored config for the same `id`. Compare `RawConfig`, `AuthContext`, and `CapabilityKey`. Return `is_duplicate: true` if they all match.

---

### 8.4 `GET /<service>/action/health`

Health check endpoint. The executor calls this every 30 seconds to verify plugin liveness.

**Response (200 OK):**
```json
{
  "status": "ok",
  "message": "service is healthy",
  "active_consumers": 3,
  "timestamp": "2026-05-16T12:00:00Z"
}
```

**Note:** If this endpoint returns a non-200 status or is unreachable, the executor marks the container status as `"error"` in its database, which prevents new workflows from being assigned to it.

---

## 9. RabbitMQ Consumption (Receiving Tasks)

### 9.1 Queue Assignment

The executor creates and manages the action queue. Your plugin does NOT create the queue topology — it only consumes from the queue name provided in the `/setup` call.

Queue naming convention: `workflow_<workflow_id>_action`

The executor handles:
1. Declaring the queue (durable)
2. Declaring the `EVENT_MESSAGE` exchange (topic)
3. Binding the queue to `EVENT_MESSAGE` with routing key = `workflow_id`

Your consumer just needs to:
1. Declare the queue (idempotent, ensures it exists)
2. Start consuming with manual acknowledgment

### 9.2 Connection

```
URL: RABBITMQ_URL env var (default: amqp://guest:guest@localhost:5672/)
```

Each `/setup` call creates a new consumer with its own connection to RabbitMQ.

### 9.3 Consumer Tag

Use a descriptive, unique tag for each consumer:
```
consumerTag = "<service>-action-<workflow_id>"
// e.g. "slack-action-abc123-def456"
```

### 9.4 Message Format

Each message on the queue is a JSON-serialized `ActionTask` (see §6.3). This is the same `TriggerEvent` published by the trigger plugin.

### 9.5 Acknowledgment Rules

| Scenario | Action |
|---|---|
| Task processed successfully | `Ack(false)` — remove from queue |
| All retries exhausted (permanent failure) | `Nack(false, false)` — reject without requeue |
| Unmarshal error (bad message) | `Nack(false, false)` — dead-letter, don't requeue |
| Auth error (cannot get valid credentials) | `Nack(false, false)` — reject without requeue |
| Unknown capability key | `Nack(false, false)` — reject without requeue |

**Critical:** Never requeue a message you cannot process. This prevents infinite retry loops that would block the queue.

---

## 10. Template Resolution

### 10.1 Overview

Trigger payloads flow into action config values through template tokens. When a user configures an action, they can use `{{trigger.payload.<key>}}` placeholders in any string config field. These are resolved at runtime when a task arrives.

**Example:**
- User configures action with: `message = "New video: {{trigger.payload.title}}"`
- Trigger fires with payload: `{ "title": "How to build a plugin" }`
- Resolved at runtime: `message = "New video: How to build a plugin"`

### 10.2 Template Pattern

```
{{trigger.payload.<field_name>}}
```

Where `<field_name>` matches `\w+` (alphanumeric + underscore).

### 10.3 Resolution Algorithm

```
resolveTemplates(cfg: ActionConfig, payload: map):
    if payload is null or cfg.RawConfig is null:
        return  // nothing to resolve

    // 1. Deep-copy RawConfig to avoid mutating the stored config.
    //    This is CRITICAL — without this copy, resolving templates for
    //    one event would corrupt the templates for future events.
    resolved = shallow copy of cfg.RawConfig

    // 2. For each string value in resolved config:
    for key, value in resolved:
        if value is a string and contains "{{trigger.payload.":
            replace all {{trigger.payload.<X>}} with payload[X]
            if payload[X] is missing, log warning and leave template as-is

    // 3. Re-extract typed fields from the resolved config
    //    (e.g. cfg.Channel = resolved["channel"])
    update service-specific fields from resolved map
```

### 10.4 Important Rules

1. **Always deep-copy before resolving.** The stored config must retain the original templates for future events.
2. **Resolve on the task-level copy, not the stored config.** Each task gets its own resolved copy.
3. **Missing keys are left unreplaced.** If the trigger payload doesn't contain a referenced key, log a warning but don't crash.
4. **Convert non-string payload values to string.** Use `fmt.Sprintf("%v", val)` or equivalent.

---

## 11. Task Execution & Capability Routing

### 11.1 Task Router Pattern

The task router is a factory function that returns a closure (callback). This pattern allows each consumer to capture its own config state.

```
HandleTaskRouter(cfgProvider, publisher, seq, instanceID, initialCfg) -> func(delivery):
    currentCfg = copy of initialCfg   // independent per-consumer state

    return func(delivery):
        // 1. Unmarshal message body → ActionTask
        task = JSON.unmarshal(delivery.body)

        // 2. Copy & resolve templates
        taskCfg = copy of currentCfg
        resolveTemplates(taskCfg, task.payload)

        // 3. Get valid auth (refresh if needed)
        auth = service.GetValidAuth(instanceID, cfgProvider, currentCfg)

        // 4. Route to capability
        capability = taskCfg.capability_key
        switch capability:
            case "slack_post_message":
                output, elapsed, err = service.PostMessage(auth, taskCfg)
            case "spotify_add_to_queue":
                output, elapsed, err = service.AddToQueue(auth, taskCfg)
            default:
                log "Unknown capability"
                delivery.Nack(false, false)
                return

        // 5. Retry loop (see §13)
        // 6. Publish result (see §12)
        // 7. Ack or Nack
```

### 11.2 Capability Implementation Pattern

Each capability function follows this signature:

```
func <CapabilityName>(auth: AuthData, cfg: ActionConfig) -> (output: map, elapsed_ms: int64, error):
    1. Validate required config fields (return error if missing)
    2. Build request to external API
    3. Set auth headers (Bearer token, API key, etc.)
    4. Execute request, measure elapsed time
    5. Parse response
    6. Check for API-level errors (not just HTTP status)
    7. Build output map matching the declared output_schema
    8. Return (output, elapsed, nil) on success
    9. Return (nil, elapsed, error) on failure
```

### 11.3 Determining the Token

Actions may receive multiple credential types. Use this fallback chain:
```
token = auth.access_token
if token is empty: token = auth.bot_token
if token is empty: token = auth.api_key
if token is empty: return error "no valid token found"
```

---

## 12. Result Publishing

### 12.1 When to Publish

Publish an `ActionResult` after every execution attempt:

| Outcome | Status | When |
|---|---|---|
| Action succeeded | `"success"` | Immediately after successful API call |
| Action failed, will retry | `"retrying"` | After each failed attempt, before sleeping |
| Action failed, no retries left | `"error"` | After the final failed attempt |

### 12.2 Exchange & Routing

```
Exchange:      "ACTION_RESULT"   (direct exchange, declared by workflow_executor)
Routing Key:   workflow_id       (string)
Content-Type:  "application/json"
Delivery Mode: Persistent (2)
Body:          JSON-serialized ActionResult (see §6.4)
```

### 12.3 Building the ActionResult

```
publishResult(publisher, task, resultOutput, elapsedMs, error, status, retryCount):
    result = ActionResult {
        task_id:        task.id,
        workflow_id:    task.workflow_id,
        timestamp:      now(UTC),
        response_time:  elapsedMs,
        status:         status,
        retry_count:    retryCount
    }

    if status == "error" and error is not nil:
        result.success = false
        result.error   = error.message

    else if status == "success":
        result.success = true
        result.output  = resultOutput

    else if status == "retrying" and error is not nil:
        result.success = false
        result.error   = error.message

    publisher.Publish(task.workflow_id, result)
```

### 12.4 What Happens After Publishing (Executor Side)

The executor runs a `MonitorService` that consumes from a per-workflow monitoring queue (bound to both `TRIGGER_RESULT` and `ACTION_RESULT` exchanges). When it receives your `ActionResult`:

1. Looks up or creates a `WorkflowJob` record using `task_id`
2. Updates the job status (`success`, `failed`, `retrying`)
3. Sets `completed_at` timestamp on terminal statuses
4. Creates a `WorkflowJobEvent` audit record with the full payload

---

## 13. Retry & Error Handling

### 13.1 Retry Loop

```
retryCount = RETRY_COUNT (default: 3)
retryDelay = RETRY_SECONDS (default: 10 seconds)

for attempt in 0..retryCount:
    output, elapsed, err = executeCapability(auth, taskCfg)

    // Handle 401 (OAuth2 token expired) — immediate refresh + retry
    if err and err contains "401":
        newAuth = service.RefreshAccessToken(auth)
        if refresh succeeded:
            update auth in currentCfg and configProvider
            auth = newAuth
            output, elapsed, err = executeCapability(auth, taskCfg)  // retry immediately

    if err is nil:
        // SUCCESS
        publishResult(publisher, task, output, elapsed, nil, "success", attempt)
        delivery.Ack(false)
        return

    if attempt < retryCount:
        // RETRYING
        publishResult(publisher, task, nil, elapsed, err, "retrying", attempt + 1)
        sleep(retryDelay)
    else:
        // PERMANENT FAILURE
        publishResult(publisher, task, nil, elapsed, err, "error", attempt)
        delivery.Nack(false, false)
```

### 13.2 Error Categories

| Error Type | Retry? | Action |
|---|---|---|
| Network error (timeout, DNS) | Yes | Retry with backoff |
| HTTP 401 Unauthorized | Special | Refresh token, retry once immediately |
| HTTP 429 Too Many Requests | Yes | Retry with backoff (respect Retry-After header if present) |
| HTTP 5xx Server Error | Yes | Retry with backoff |
| HTTP 400 Bad Request | No | Permanent failure (bad config/data) |
| HTTP 403 Forbidden | No | Permanent failure (insufficient permissions) |
| Unknown capability key | No | Nack immediately |
| JSON unmarshal error | No | Nack immediately |
| Missing auth credentials | No | Nack immediately |

### 13.3 Retry Configuration

| Variable | Default | Description |
|---|---|---|
| `RETRY_COUNT` | `3` | Maximum number of retry attempts after initial failure |
| `RETRY_SECONDS` | `10` | Seconds to wait between retries |

---

## 14. Authentication & Token Refresh

### 14.1 Non-Expiring Tokens (Slack Bot Tokens, API Keys)

For services that use non-expiring tokens, the auth helper is trivial:

```
GetValidAuth(id, cfgProvider, cfg) -> (AuthData, error):
    return getFirstAuthEntry(cfg.AuthContext)
```

No refresh logic needed.

### 14.2 Expiring OAuth2 Tokens (Spotify, Google, etc.)

For services with expiring access tokens, implement proactive refresh:

```
GetValidAuth(id, cfgProvider, cfg) -> (AuthData, error):
    auth = getFirstAuthEntry(cfg.AuthContext)

    if auth.access_token is empty:
        return error "no auth data found"

    // Check if token is expired or expiring within 5 minutes
    if auth.expiry is not zero and now + 5min > auth.expiry:
        newAuth = RefreshAccessToken(auth)
        // Update the config's auth context
        for each key in cfg.AuthContext:
            cfg.AuthContext[key] = newAuth
        cfgProvider.UpdateAuth(id, cfg.AuthContext)
        return newAuth

    return auth
```

### 14.3 Token Refresh Implementation

```
RefreshAccessToken(auth) -> (AuthData, error):
    POST to <service_token_url> with:
        grant_type = "refresh_token"
        refresh_token = auth.refresh_token
        client_id = <SERVICE>_CLIENT_ID (env var)
        client_secret = <SERVICE>_CLIENT_SECRET (env var)
    
    Parse response:
        access_token, expires_in, refresh_token (optional new one)
    
    Update auth fields:
        auth.access_token = new_access_token
        if new_refresh_token is not empty:
            auth.refresh_token = new_refresh_token
        auth.expiry = now + expires_in seconds
    
    return auth
```

### 14.4 Reactive 401 Refresh

In addition to proactive refresh, handle 401 errors reactively during the retry loop (see §13.1). This catches edge cases where the token expired between the check and the API call.

---

## 15. Environment Variables

| Variable | Default | Description |
|---|---|---|
| `EXECUTOR_URL` | `http://localhost:8082` | Workflow executor base URL |
| `PLUGIN_HOST` | `localhost` | Hostname the executor uses to reach this plugin |
| `PLUGIN_PORT` | `8086` | Advertised port for registration |
| `PLUGIN_LISTEN_PORT` | (none) | Internal listen port (for Nginx proxying in unified container) |
| `RABBITMQ_URL` | `amqp://guest:guest@localhost:5672/` | RabbitMQ connection string |
| `HOSTNAME` | (set by Docker) | Container hostname, used for plugin ID |
| `HTTP_PROXY` | (none) | HTTP proxy for outbound API calls |
| `HTTPS_PROXY` | (none) | HTTPS proxy for outbound API calls |
| `RETRY_COUNT` | `3` | Maximum retry attempts |
| `RETRY_SECONDS` | `10` | Seconds between retries |

Add service-specific env vars as needed (e.g. `SPOTIFY_CLIENT_ID`, `SPOTIFY_CLIENT_SECRET`, `SLACK_BOT_TOKEN`).

---

## 16. Deployment

### 16.1 Standalone Dockerfile

```dockerfile
FROM <language-base-image> AS builder
WORKDIR /app
COPY . .
RUN <build-command> -o plugin_bin .

FROM alpine:latest
RUN apk --no-cache add ca-certificates
WORKDIR /app
COPY --from=builder /app/plugin_bin .

ENV RABBITMQ_URL="amqp://guest:guest@localhost:5672/"
ENV EXECUTOR_URL="http://localhost:8082"
ENV PLUGIN_HOST="localhost"
ENV PLUGIN_PORT="8086"

EXPOSE 8086
CMD ["./plugin_bin"]
```

### 16.2 Unified Container (Production)

All plugins run in a single container behind Nginx:

1. **Add your plugin to `active_plugins.txt`:**
   ```
   <service>_action
   ```

2. **Add build steps to the root `Dockerfile`:**
   ```dockerfile
   COPY <service>_action/ ./<service>_action/
   RUN cd <service>_action && <build-command> -o /app/<service>_action_bin .
   ```

3. **Copy the binary in the runtime stage:**
   ```dockerfile
   COPY --from=builder /app/<service>_action_bin .
   ```

4. **`start.sh` auto-assigns ports** from `active_plugins.txt`:
   - Reads each line, derives the binary name (`<service>_action_bin`) and route prefix (`/<service>/action`)
   - Assigns sequential ports starting from 8081
   - Generates Nginx reverse-proxy config
   - Starts each binary with `PLUGIN_LISTEN_PORT=<assigned_port>`
   - Nginx listens on port 7860 (the public-facing port)

5. **Port assignment:** Plugins are assigned ports sequentially based on their position in `active_plugins.txt`. The first plugin gets 8081, the second gets 8082, etc. The `PLUGIN_HOST` and `PLUGIN_PORT` for registration should reflect the external-facing address (Nginx on port 7860, or the Hugging Face Spaces URL).

---

## 17. End-to-End Walkthrough

Here is the complete lifecycle for a hypothetical **"Discord" action plugin**:

### Step 1: Create directory structure

```
discord_action/
├── main.go
├── go.mod
├── models/
│   └── models.go
├── services/
│   ├── registration.go
│   └── discord.go
└── worker/
    ├── consumer.go
    └── publisher.go
```

### Step 2: Define models (`models/models.go`)

```
ActionConfig:
    capability_key: string
    auth_context:   map[string]AuthData
    raw_config:     map[string]interface{}   // preserved for template resolution
    // Service-specific fields:
    channel_id:     string                   // Discord channel ID
    message:        string                   // Message content (supports templates)
    embed_title:    string                   // Optional embed title
    embed_url:      string                   // Optional embed URL

AuthData:            (standard, see §6.2)
ActionTask:          (standard, see §6.3)
ActionResult:        (standard, see §6.4)
RegistrationRequest: (standard, see §6.5)
PluginCapability:    (standard, see §6.6)
```

### Step 3: Define registration (`services/registration.go`)

```
POST to {EXECUTOR_URL}/register with:
{
  "id": "{HOSTNAME}-discord-action",
  "name": "Discord Action",
  "container_type": "action",
  "plugin_provider_service": "Discord",
  "plugin_host": "{PLUGIN_HOST}",
  "plugin_port": "{PLUGIN_PORT}",
  "endpoints": {
    "setup":  "/discord/action/setup",
    "remove": "/discord/action/remove",
    "health": "/discord/action/health"
  },
  "auth_types": ["Bot Token"],
  "capabilities": [
    {
      "unique_key": "discord_send_message",
      "name": "Send a message to a channel",
      "description": "Sends a text message to the specified Discord channel.",
      "component_type": "ACTION",
      "config_schema": {
        "channel_id": {
          "type": "string",
          "required": true,
          "description": "Discord channel ID to send the message to"
        },
        "message": {
          "type": "string",
          "required": true,
          "description": "Message text. Supports {{trigger.payload.X}} templates."
        }
      },
      "output_schema": {
        "message_id": "string",
        "channel_id": "string",
        "timestamp": "string"
      }
    },
    {
      "unique_key": "discord_send_embed",
      "name": "Send an embed to a channel",
      "description": "Sends a rich embed message to the specified Discord channel.",
      "component_type": "ACTION",
      "config_schema": {
        "channel_id": {
          "type": "string",
          "required": true,
          "description": "Discord channel ID"
        },
        "embed_title": {
          "type": "string",
          "required": true,
          "description": "Title for the embed"
        },
        "message": {
          "type": "string",
          "required": false,
          "description": "Description text for the embed"
        },
        "embed_url": {
          "type": "string",
          "required": false,
          "description": "URL to link in the embed"
        }
      },
      "output_schema": {
        "message_id": "string",
        "channel_id": "string",
        "timestamp": "string"
      }
    }
  ]
}
```

### Step 4: Implement service logic (`services/discord.go`)

```
DiscordService struct:
    httpClient, retrySeconds, retryCount

NewDiscordService() -> *DiscordService
    (configure HTTP client with proxy support, IPv4-only)

resolveTemplates(cfg, payload):
    deep-copy RawConfig
    resolve {{trigger.payload.X}} patterns in all string values
    re-extract: cfg.ChannelID, cfg.Message, cfg.EmbedTitle, cfg.EmbedURL

GetValidAuth(id, cfgProvider, cfg) -> (AuthData, error):
    auth = getFirstAuthEntry(cfg.AuthContext)
    // Discord bot tokens don't expire — return directly
    token = auth.bot_token or auth.access_token or auth.api_key
    if token is empty: return error
    return auth

publishResult(publisher, task, output, elapsed, err, status, retry):
    // Standard result publishing (see §12.3)

HandleTaskRouter(cfgProvider, publisher, seq, instanceID, initialCfg) -> func(delivery):
    currentCfg = initialCfg

    return func(delivery):
        task = unmarshal(delivery.body) as ActionTask
        taskCfg = copy of currentCfg
        resolveTemplates(taskCfg, task.payload)

        auth = GetValidAuth(instanceID, cfgProvider, taskCfg)
        capability = taskCfg.capability_key

        for attempt in 0..retryCount:
            switch capability:
                case "discord_send_message":
                    output, elapsed, err = SendMessage(auth, taskCfg)
                case "discord_send_embed":
                    output, elapsed, err = SendEmbed(auth, taskCfg)
                default:
                    log "Unknown capability"
                    delivery.Nack(false, false)
                    return

            if err is nil:
                publishResult(publisher, task, output, elapsed, nil, "success", attempt)
                delivery.Ack(false)
                return

            if attempt < retryCount:
                publishResult(publisher, task, nil, elapsed, err, "retrying", attempt+1)
                sleep(retrySeconds)
            else:
                publishResult(publisher, task, nil, elapsed, err, "error", attempt)
                delivery.Nack(false, false)

SendMessage(auth, cfg) -> (output, elapsed, error):
    if cfg.ChannelID is empty: return error "channel_id required"
    if cfg.Message is empty: return error "message required"

    token = auth.bot_token or auth.access_token
    endpoint = "https://discord.com/api/v10/channels/{cfg.ChannelID}/messages"

    POST endpoint with:
        Authorization: "Bot {token}"
        Content-Type: "application/json"
        Body: { "content": cfg.Message }

    Parse response → extract message_id, channel_id, timestamp
    return output map, elapsed, nil

SendEmbed(auth, cfg) -> (output, elapsed, error):
    // Similar pattern with embed object in request body
```

### Step 5: Implement consumer (`worker/consumer.go`)

Standard RabbitMQ consumer (see §5.5).

### Step 6: Implement publisher (`worker/publisher.go`)

Standard RabbitMQ publisher for ActionResults (see §5.6 and §12).

### Step 7: Wire up main.go

```
func main():
    discordSvc = NewDiscordService()
    go registerWithRetry()

    prefix = "/discord/action"
    route(prefix + "/setup",    handleSetup)
    route(prefix + "/remove",   handleRemove)
    route(prefix + "/validate", handleValidate)
    route(prefix + "/health",   handleHealth)

    listen(getPort())
```

### Step 8: Deploy

Add `discord_action` to `active_plugins.txt`, add build steps to unified `Dockerfile`, push.

---

## 18. Quick Reference Tables

### Endpoint Summary

| Endpoint | Method | Request Body | Response |
|---|---|---|---|
| `POST /register` (on executor) | POST | `RegistrationRequest` | 2xx = success |
| `/<svc>/action/setup` | POST | `SetupPayload` | `{"success":true,"message":"setup_complete","queue_name":"...","seq":N}` |
| `/<svc>/action/remove` | POST | `RemovePayload` | `{"success":true,"message":"removed","removed_count":N}` |
| `/<svc>/action/validate` | POST | `SetupPayload` | `{"is_duplicate":bool}` |
| `/<svc>/action/health` | GET | — | `{"status":"ok","active_consumers":N,"timestamp":"..."}` |

### RabbitMQ Topology

| Component | Name | Type | Purpose |
|---|---|---|---|
| Queue (consume) | `workflow_<id>_action` | Durable | Receives trigger events for this workflow |
| Exchange (consume) | `EVENT_MESSAGE` | Topic | Routes trigger events to action queues |
| Exchange (publish) | `ACTION_RESULT` | Direct | Routes action results to monitoring queue |
| Routing key (consume) | `workflow_id` | — | Binds queue to exchange |
| Routing key (publish) | `workflow_id` | — | Routes result to correct monitoring queue |

### Data Flow Summary

```
Trigger Plugin                 RabbitMQ                     Action Plugin
     │                            │                              │
     │  TriggerEvent              │                              │
     ├──────────────────────────► │                              │
     │  Exchange: EVENT_MESSAGE   │  Queue: workflow_<id>_action │
     │  Key: workflow_id          ├─────────────────────────────►│
     │                            │                              │
     │                            │  1. Unmarshal ActionTask     │
     │                            │  2. Resolve templates        │
     │                            │  3. Get valid auth           │
     │                            │  4. Route to capability      │
     │                            │  5. Call external API ───────┼──► External Service
     │                            │  6. Build ActionResult       │◄── Response
     │                            │                              │
     │                            │  ActionResult                │
     │                            │◄─────────────────────────────┤
     │  Exchange: ACTION_RESULT   │                              │
     │  Key: workflow_id          │                              │
     │                            │                              │
     │                     ┌──────┤                              │
     │                     │ Monitoring Queue (workflow_id)      │
     │                     │ → Executor MonitorService           │
     │                     │   updates WorkflowJob status        │
     │                     └──────┘                              │
```

---

## 19. Design Principles

1. **One consumer per action instance** — each `/setup` creates an isolated consumer with its own config copy. Multiple workflows run without interference.
2. **Idempotent re-setup** — calling `/setup` with the same `id` stops the existing consumer before creating a new one.
3. **Graceful removal** — `/remove` supports both ID-based and workflow-based lookup for flexibility.
4. **Lazy reconnection** — the RabbitMQ consumer reconnects transparently when a connection is lost. The publisher reconnects on the next publish attempt.
5. **Template immutability** — the stored `ActionConfig.RawConfig` always retains the original `{{trigger.payload.X}}` templates. Resolution happens on a per-task copy.
6. **Proxy support** — honor `HTTP_PROXY`/`HTTPS_PROXY` env vars for outbound API calls (important for hosted environments with IP restrictions).
7. **Thread safety** — use mutex for the consumers map, thread-safe map for configs, and atomic operations for counters.
8. **Output schema contract** — `ActionResult.output` keys SHOULD match the `output_schema` declared at registration.
9. **Result publishing is mandatory** — every execution attempt (success, retry, or failure) MUST publish an `ActionResult` so the executor can track job status.
10. **Auth state independence** — each consumer maintains its own copy of auth credentials. Token refreshes update the consumer's local state without affecting other consumers.
