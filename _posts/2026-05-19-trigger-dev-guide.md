---
layout: post
title: "Sequels.diy compatible Trigger development guide"
date: 2026-05-19
categories: sequels trigger dev-guide
---
This guide provides a concise, language-agnostic specification and implementation blueprint for building Sequels.diy-compatible trigger plugins. It covers architecture, data models, HTTP endpoints, RabbitMQ publishing, polling strategies, and deployment patterns so you can implement reliable triggers in any language that supports HTTP and Rabbit MQ AMQP.

> **Language-agnostic specification.** All examples use JSON for data structures. Implement in any language that can serve HTTP and connect to RabbitMQ (AMQP 0-9-1).

---

## Table of Contents

1. [Pre-Development Checklist](#1-pre-development-checklist)
2. [Architecture Overview](#2-architecture-overview)
3. [Directory Structure](#3-directory-structure)
4. [File-by-File Specification](#4-file-by-file-specification)
5. [Data Models](#5-data-models)
6. [Workflow Executor Registration](#6-workflow-executor-registration)
7. [HTTP Endpoints](#7-http-endpoints)
8. [RabbitMQ Event Publishing](#8-rabbitmq-event-publishing)
9. [Polling / Scheduling Loop](#9-polling--scheduling-loop)
10. [Environment Variables](#10-environment-variables)
11. [Deployment](#11-deployment)
12. [End-to-End Walkthrough](#12-end-to-end-walkthrough)

---

## 1. Pre-Development Checklist

Before writing any code, confirm every item below:

- [ ] **Plugin name** decided — format: `<service>_trigger` (e.g. `github_trigger`, `rss_trigger`)
- [ ] **Route prefix** decided — format: `/<service>/trigger` (e.g. `/github/trigger`)
- [ ] **Capabilities list** finalized — each capability needs:
  - [ ] `unique_key` — globally unique string (e.g. `github_new_issue`)
  - [ ] `name` — human-readable label
  - [ ] `description` — one-sentence explanation
  - [ ] `config_schema` — map of parameter names → types the user must supply
  - [ ] `output_schema` — map of payload field names → types emitted when the trigger fires
- [ ] **Auth type** decided — one of: `"None"`, `"OAuth 2.0"`, `"Personal Access Token"`, `"API Key"`, `"Bot Token"`
- [ ] **Polling strategy** decided — interval-based polling or time-schedule matching or some other trigger method
- [ ] **Deduplication strategy** decided — cache-based (seen-set) or time-based (`lastCheck` timestamp)
- [ ] **RabbitMQ access** confirmed — connection string available
- [ ] **Workflow executor URL** known — default `http://localhost:8082`
- [ ] **Directory structure** created (see §3)
- [ ] **Dockerfile** written (standalone) or entry added to unified `Dockerfile` + `active_plugins.txt`
- [ ] **Environment variables** documented

---

## 2. Architecture Overview

```
                        ┌──────────────────────────┐
                        │    workflow_executor      │
                        │  (orchestration backend)  │
                        └──────┬───────────▲────────┘
                    POST /register     POST /<svc>/trigger/setup
                    (on startup)       POST /<svc>/trigger/remove
                               │               │
                        ┌──────▼───────────────┐
                        │   YOUR TRIGGER PLUGIN │
                        │  HTTP server on :PORT  │
                        └──────────┬────────────┘
                                   │ poll / schedule
                                   │ (goroutine per instance)
                                   ▼
                        ┌──────────────────────┐
                        │   External Service   │
                        │  (GitHub, YouTube…)   │
                        └──────────┬───────────┘
                                   │ new data detected
                                   ▼
                        ┌──────────────────────┐
                        │      RabbitMQ         │
                        │ Exchange: EVENT_MESSAGE│
                        │ Routing key: workflow_id│
                        └───────────────────────┘
```

**Lifecycle:**
1. Plugin starts → connects to RabbitMQ → registers with `workflow_executor` via `POST /register`
2. Executor calls `POST /<svc>/trigger/setup` with config → plugin starts a poller/scheduler goroutine
3. Poller detects new data → publishes `TriggerEvent` to RabbitMQ
4. Executor calls `POST /<svc>/trigger/remove` → plugin stops the poller and cleans up

---

## 3. Directory Structure

```
<service>_trigger/
├── main.go                    # HTTP server, route registration, global state, handlers
├── go.mod                     # Module definition (or equivalent for your language)
├── go.sum                     # Dependency lock file
├── Dockerfile                 # Standalone container build (optional)
├── models/
│   └── models.go              # All data structures: TriggerConfig, AuthData, TriggerEvent,
│                              #   RegistrationRequest, PluginCapability
├── services/
│   ├── registration.go        # Self-registration logic (POST to workflow_executor /register)
│   └── <service>.go           # Polling/scheduling logic: Poller/Scheduler struct,
│                              #   Start(), Stop(), poll()/check(), per-capability handlers
└── worker/
    └── publisher.go           # RabbitMQ connection management + Publish() method
```

---

## 4. File-by-File Specification

### 4.1 `main.go` — Entry Point & HTTP Server

**Responsibilities:**
- Declare global state: RabbitMQ publisher, poller/scheduler map, config store (thread-safe), mutex
- Define `SetupPayload` and `RemovePayload` structs for JSON decoding
- Implement `buildTriggerConfig()` to extract typed config from raw JSON map
- Implement HTTP handlers: `handleSetup`, `handleRemove`, `handleValidate`, `handleHealth`
- On startup: initialize publisher, start background registration with retry, register HTTP routes, listen on port

**Global state pattern:**
```
publisher       — singleton RabbitMQ publisher
pollers         — map[string]*Poller   (keyed by trigger instance ID)
configs         — thread-safe map      (keyed by trigger instance ID)
mu              — mutex for pollers map
pollerCount     — atomic counter for logging
```

**Port resolution order:** `PLUGIN_LISTEN_PORT` → `PLUGIN_PORT` → `"8080"` (default)

**Registration retry:** Up to 10 attempts with linear backoff (`(attempt) * 5 seconds`), run in a background thread so the HTTP server starts immediately.

---

### 4.2 `models/models.go` — Data Structures

Defines all shared types. See §5 for full schemas.

---

### 4.3 `services/registration.go` — Executor Registration

**Responsibilities:**
- Build a `RegistrationRequest` with plugin metadata and capabilities list
- POST it as JSON to `{EXECUTOR_URL}/register`
- Build a unique plugin ID: `{HOSTNAME}-<service>-trigger` (HOSTNAME is set by Docker; fallback to `{host}:{port}-<service>-trigger`)

---

### 4.4 `services/<service>.go` — Polling / Scheduling Logic

**Responsibilities:**
- Define a `Poller` (or `Scheduler` for time-based triggers) struct holding: trigger ID, workflow ID, config, auth tokens, HTTP client, stop channel, dedup state
- `NewPoller()` — extract auth credentials from `_auth_context`, configure HTTP client (with proxy support via `HTTP_PROXY`/`HTTPS_PROXY` env vars)
- `Start()` — launch background goroutine with a ticker (e.g. every 2 minutes for API polling, every 1 minute for time-based)
- `Stop()` — close stop channel (use once-guard to prevent double-close)
- `poll()` / `check()` — dispatch to per-capability handler, apply dedup, call `fireEvent()` for new items
- Per-capability functions — call external API, parse response, return normalized `[]map[string]interface{}`
- `fireEvent()` — build `TriggerEvent`, call `publisher.Publish(workflowID, event)`

**Dedup strategies:**
- **Cache-based** (recommended): maintain an ordered list + set of seen IDs. On first poll, seed the cache and optionally fire for the latest item only. On subsequent polls, fire only for IDs not in the set. Evict oldest entries when cache exceeds ~100 items.
- **Time-based** (simpler): store `lastCheck` timestamp, only fire for items created after it, advance after each poll.

---

### 4.5 `worker/publisher.go` — RabbitMQ Publisher

**Responsibilities:**
- Connect to RabbitMQ using `RABBITMQ_URL` env var (default: `amqp://guest:guest@localhost:5672/`)
- Maintain connection + channel with lazy reconnection (`ensureConnected()`)
- `Publish(workflowID, event)` — JSON-marshal the `TriggerEvent`, publish to exchange `"EVENT_MESSAGE"` with routing key = `workflowID`, content type `application/json`, delivery mode persistent
- Thread-safe via mutex

---

## 5. Data Models

### 5.1 `TriggerConfig`

Holds all configuration for one trigger instance.

```json
{
  "capability_key": "string",
  "_auth_context": {
    "<provider_name>": { AuthData }
  },
  // ...service-specific fields (see below)
}
```

**Service-specific fields examples:**
| Service | Fields |
|---|---|
| GitHub | `repository` (string), `username_or_organization` (string) |
| YouTube | `search_query` (string), `channel_name_or_id` (string) |
| RSS | `feed_url` (string), `keyword` (string) |
| Instagram | `hashtag` (string) |
| Datetime | `scheduled_at` (string), `day_of_week` (string), `day_of_month` (int) |

### 5.2 `AuthData`

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

### 5.3 `TriggerEvent`

Published to RabbitMQ when the trigger fires.

```json
{
  "id": "uuid-v4",
  "workflow_id": "string",
  "trigger_id": "string",
  "type": "event",
  "name": "Human-Readable Trigger Name",
  "capability_key": "string",
  "payload": {
    // keys must match the output_schema declared at registration
  },
  "timestamp": "ISO8601 datetime (UTC)"
}
```

**Critical:** The keys inside `payload` MUST exactly match the `output_schema` keys declared during registration. The `workflow_executor` uses these keys to resolve `{{trigger.payload.<key>}}` template tokens in downstream action configs.

### 5.4 `RegistrationRequest`

Sent to `workflow_executor` at startup.

```json
{
  "id": "unique-plugin-instance-id",
  "name": "Human-Readable Plugin Name",
  "container_type": "trigger",
  "plugin_provider_service": "ServiceName",
  "plugin_host": "hostname-or-ip",
  "plugin_port": "port-string",
  "endpoints": {
    "setup":  "/<service>/trigger/setup",
    "remove": "/<service>/trigger/remove",
    "health": "/<service>/trigger/health"
  },
  "auth_types": ["None" | "OAuth 2.0" | "Personal Access Token" | ...],
  "capabilities": [ PluginCapability, ... ]
}
```

### 5.5 `PluginCapability`

```json
{
  "unique_key": "service_capability_name",
  "name": "Human-Readable Name",
  "description": "One-sentence description.",
  "component_type": "TRIGGER",
  "config_schema": {
    "param_name": "type_string"
  },
  "output_schema": {
    "output_field": "type_string"
  }
}
```

- `config_schema`: keys = config parameter names the user supplies. Values = type hints (`"string"`, `"integer"`). An empty map `{}` means no user config needed.
- `output_schema`: keys = payload field names emitted on trigger fire. These become available as `{{trigger.payload.<key>}}` in workflow action configs.

---

## 6. Workflow Executor Registration

### 6.1 When

Immediately after the HTTP server is ready to accept requests. Run in a background thread so the server isn't blocked.

### 6.2 Endpoint

```
POST {EXECUTOR_URL}/register
Content-Type: application/json
Body: RegistrationRequest (see §5.4)
```

### 6.3 Response

- **2xx** — success, plugin is registered
- **Non-2xx** — retry with backoff

### 6.4 Retry Strategy

```
for attempt in 1..10:
    response = POST /register
    if response.status is 2xx:
        return success
    wait (attempt * 5) seconds
log warning: "Could not register after 10 attempts"
```

### 6.5 Plugin ID Construction

Since all plugins may run inside a single container sharing the same `HOSTNAME` env var, each plugin MUST append a unique suffix:

```
id = HOSTNAME + "-<service>-trigger"    // e.g. "abc123-github-trigger"
```

Fallback when HOSTNAME is empty:
```
id = PLUGIN_HOST + ":" + PLUGIN_PORT + "-<service>-trigger"
```

---

## 7. HTTP Endpoints

All endpoints use the prefix `/<service>/trigger` (e.g. `/github/trigger`).

### 7.1 `POST /<service>/trigger/setup`

Called by `workflow_executor` to activate a trigger instance.

**Request Body:**
```json
{
  "id": "trigger-instance-uuid",
  "trigger_id": "trigger-instance-uuid",
  "workflow_id": "workflow-uuid",
  "queue_name": "optional-queue-name",
  "capability_key": "service_capability_name",
  "config": {
    "param1": "value1",
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
- Use `id` as the primary identifier. Fall back to `trigger_id` if `id` is empty.
- `_auth_context` is injected by `workflow_executor` with decrypted user credentials. It is a map keyed by provider name.
- If a poller already exists for this `id`, stop it before creating a new one (idempotent re-setup).

**Response (200 OK):**
```json
{
  "status": "setup_complete"
}
```

**Error (400 Bad Request):** returned if JSON is malformed.

---

### 7.2 `POST /<service>/trigger/remove`

Called by `workflow_executor` to deactivate a trigger instance.

**Request Body:**
```json
{
  "id": "trigger-instance-uuid",
  "trigger_id": "trigger-instance-uuid",
  "workflow_id": "workflow-uuid"
}
```

**Removal logic (in order):**
1. If `id` (or `trigger_id`) is provided, look up and stop that specific poller.
2. If `workflow_id` is provided, iterate all pollers and stop any matching that workflow (bulk removal fallback).
3. Clean up both the poller map and the config store.

**Response (200 OK):**
```json
{
  "status": "removed",
  "removed_count": 1
}
```

---

### 7.3 `POST /<service>/trigger/validate` *(optional but recommended)*

Called to check if an identical trigger is already running (duplicate detection).

**Request Body:** Same as `/setup`.

**Response (200 OK):**
```json
{
  "is_duplicate": true
}
```

**Logic:** Build a `TriggerConfig` from the payload, compare it field-by-field (or via deep-equal) with the stored config for the same `id`. Return `is_duplicate: true` if they match.

---

### 7.4 `GET /<service>/trigger/health`

Health check endpoint.

**Response (200 OK):**
```json
{
  "status": "ok",
  "message": "healthy",
  "active_triggers": 3,
  "timestamp": "2026-05-16T12:00:00Z"
}
```

---

## 8. RabbitMQ Event Publishing

### 8.1 Connection

```
URL: RABBITMQ_URL env var (default: amqp://guest:guest@localhost:5672/)
```

Connect on startup. If connection fails, log a warning and retry lazily on first publish.

### 8.2 Publishing

```
Exchange:      "EVENT_MESSAGE"   (topic exchange, declared by workflow_executor)
Routing Key:   workflow_id       (string)
Content-Type:  "application/json"
Delivery Mode: Persistent (2)
Body:          JSON-serialized TriggerEvent (see §5.3)
```

### 8.3 Reconnection

Before each publish, check if the connection is alive. If not, reconnect. If publish fails, invalidate the connection so the next attempt reconnects.

---

## 9. Polling / Scheduling Loop

### 9.1 API-Polling Triggers (GitHub, YouTube, Instagram, RSS)

- **Interval:** Typically every 2 minutes (configurable per plugin).
- **First poll:** Immediately on setup, then on ticker.
- **Stop:** Select on stop channel alongside ticker; return from goroutine when stop fires.

```
Start():
    go func():
        poll()                    // immediate first poll
        ticker = every 2 minutes
        loop:
            select:
                ticker -> poll()
                stopChan -> return
```

### 9.2 Time-Schedule Triggers (Datetime)

- **Interval:** Every 1 minute, aligned to `:00` seconds.
- **Alignment:** On start, wait until the next whole minute before beginning the ticker.
- **Check:** Compare current UTC time against the configured schedule.

```
Start():
    go func():
        wait until next :00 second
        ticker = every 1 minute
        check(now)                // immediate first check
        loop:
            select:
                ticker -> check(t)
                stopChan -> return
```

### 9.3 Deduplication

**Cache-based (recommended for API-polling):**
```
seenCache: ordered list of IDs (oldest → newest)
seenSet:   set of IDs for O(1) lookup

On first poll:
    seed seenCache/seenSet with all returned item IDs
    optionally fire event for the latest item (index 0)

On subsequent polls:
    for each item:
        if item._id not in seenSet:
            fireEvent(item)
            add to seenCache and seenSet
    evict oldest 10 entries if cache > 100

Each poll function must return items with an "_id" field for dedup.
The "_id" is internal and stripped from the published payload.
```

**Time-based (simpler, for legacy capabilities):**
```
lastCheck: timestamp of last poll (initialized to now on setup)

On each poll:
    fetch items from API
    for each item:
        if item.created_at > lastCheck:
            fireEvent(item)
    lastCheck = now
```

---

## 10. Environment Variables

| Variable | Default | Description |
|---|---|---|
| `EXECUTOR_URL` | `http://localhost:8082` | Workflow executor base URL |
| `PLUGIN_HOST` | `localhost` | Hostname the executor uses to reach this plugin |
| `PLUGIN_PORT` | `8085` | Advertised port for registration |
| `PLUGIN_LISTEN_PORT` | (none) | Internal listen port (for Nginx proxying in unified container) |
| `RABBITMQ_URL` | `amqp://guest:guest@localhost:5672/` | RabbitMQ connection string |
| `HOSTNAME` | (set by Docker) | Container hostname, used for plugin ID |
| `HTTP_PROXY` | (none) | HTTP proxy for outbound API calls |
| `HTTPS_PROXY` | (none) | HTTPS proxy for outbound API calls |

Add service-specific env vars as needed (e.g. `GITHUB_CLIENT_ID`, `YOUTUBE_CLIENT_SECRET`).

---

## 11. Deployment

### 11.1 Standalone Dockerfile

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
ENV PLUGIN_PORT="8085"

EXPOSE 8085
CMD ["./plugin_bin"]
```

### 11.2 Unified Container (Production)

All plugins run in a single container behind Nginx:

1. **Add your plugin to `active_plugins.txt`:**
   ```
   <service>_trigger
   ```

2. **Add build steps to the root `Dockerfile`:**
   ```dockerfile
   COPY <service>_trigger/ ./<service>_trigger/
   RUN cd <service>_trigger && <build-command> -o /app/<service>_trigger_bin .
   ```

3. **Copy the binary in the runtime stage:**
   ```dockerfile
   COPY --from=builder /app/<service>_trigger_bin .
   ```

4. **`start.sh` auto-assigns ports** from `active_plugins.txt`:
   - Reads each line, derives the binary name (`<service>_trigger_bin`) and route prefix (`/<service>/trigger`)
   - Assigns sequential ports starting from 8081
   - Generates Nginx reverse-proxy config
   - Starts each binary with `PLUGIN_LISTEN_PORT=<assigned_port>`
   - Nginx listens on port 7860 (the public-facing port)

---

## 12. End-to-End Walkthrough

Here is the complete lifecycle for a hypothetical **"Hacker News" trigger plugin**:

### Step 1: Create directory structure

```
hackernews_trigger/
├── main.go
├── go.mod
├── models/
│   └── models.go
├── services/
│   ├── registration.go
│   └── hackernews.go
└── worker/
    └── publisher.go
```

### Step 2: Define models (`models/models.go`)

```
TriggerConfig:
    capability_key: string
    auth_context:   map[string]AuthData   // empty for HN (no auth)
    min_score:      int                   // service-specific

AuthData:          (standard, see §5.2)
TriggerEvent:      (standard, see §5.3)
RegistrationRequest: (standard, see §5.4)
PluginCapability:    (standard, see §5.5)
```

### Step 3: Define registration (`services/registration.go`)

```
POST to {EXECUTOR_URL}/register with:
{
  "id": "{HOSTNAME}-hackernews-trigger",
  "name": "Hacker News Trigger",
  "container_type": "trigger",
  "plugin_provider_service": "Hacker News",
  "plugin_host": "{PLUGIN_HOST}",
  "plugin_port": "{PLUGIN_PORT}",
  "endpoints": {
    "setup":  "/hackernews/trigger/setup",
    "remove": "/hackernews/trigger/remove",
    "health": "/hackernews/trigger/health"
  },
  "auth_types": ["None"],
  "capabilities": [
    {
      "unique_key": "hn_new_top_story",
      "name": "New top story",
      "description": "Triggers when a new story hits the HN front page.",
      "component_type": "TRIGGER",
      "config_schema": {
        "min_score": "integer"
      },
      "output_schema": {
        "title": "string",
        "url": "string",
        "score": "string",
        "author": "string",
        "posted_at": "string"
      }
    }
  ]
}
```

### Step 4: Implement polling (`services/hackernews.go`)

```
Poller struct:
    triggerID, workflowID, config, publisher, stopChan, seenSet, seenCache

NewPoller(id, workflowID, config, seq, publisher) -> *Poller

Start():
    poll() immediately
    ticker every 2 minutes
    select on ticker vs stopChan

poll():
    items = fetchTopStories()   // call HN API
    deduplicate using seenSet
    for new items: fireEvent(item)

fireEvent(payload):
    event = TriggerEvent{
        id: uuid(),
        workflow_id: workflowID,
        trigger_id: triggerID,
        type: "event",
        name: "Hacker News Trigger",
        capability_key: config.capabilityKey,
        payload: {
            "title":     item.title,
            "url":       item.url,
            "score":     item.score,
            "author":    item.author,
            "posted_at": item.time
        },
        timestamp: now()
    }
    publisher.Publish(workflowID, event)
```

### Step 5: Implement publisher (`worker/publisher.go`)

Standard RabbitMQ publisher (see §4.5 and §8).

### Step 6: Wire up main.go

```
func main():
    publisher = NewPublisher()
    go registerWithRetry()

    prefix = "/hackernews/trigger"
    route(prefix + "/setup",    handleSetup)
    route(prefix + "/remove",   handleRemove)
    route(prefix + "/validate", handleValidate)
    route(prefix + "/health",   handleHealth)

    listen(getPort())
```

### Step 7: Deploy

Add `hackernews_trigger` to `active_plugins.txt`, add build steps to unified `Dockerfile`, push.

---

## Quick Reference: Endpoint Summary

| Endpoint | Method | Request Body | Response |
|---|---|---|---|
| `POST /register` (on executor) | POST | `RegistrationRequest` | 2xx = success |
| `/<svc>/trigger/setup` | POST | `SetupPayload` | `{"status":"setup_complete"}` |
| `/<svc>/trigger/remove` | POST | `RemovePayload` | `{"status":"removed","removed_count":N}` |
| `/<svc>/trigger/validate` | POST | `SetupPayload` | `{"is_duplicate":bool}` |
| `/<svc>/trigger/health` | GET | — | `{"status":"ok","active_triggers":N,"timestamp":"..."}` |

## Quick Reference: RabbitMQ Publish

| Property | Value |
|---|---|
| Exchange | `EVENT_MESSAGE` |
| Routing Key | `workflow_id` |
| Content-Type | `application/json` |
| Delivery Mode | Persistent |
| Body | `TriggerEvent` JSON |

---

## Design Principles

1. **One goroutine per trigger instance** — each `/setup` creates an isolated poller/scheduler. Multiple workflows run without interference.
2. **Idempotent re-setup** — calling `/setup` with the same `id` stops the existing poller before creating a new one.
3. **Graceful removal** — `/remove` supports both ID-based and workflow-based lookup for flexibility.
4. **Lazy reconnection** — the RabbitMQ publisher reconnects transparently on the next publish attempt.
5. **Proxy support** — honor `HTTP_PROXY`/`HTTPS_PROXY` env vars for outbound API calls (important for hosted environments with IP restrictions).
6. **Thread safety** — use mutex for the pollers map and atomic operations for counters.
7. **Output schema contract** — payload keys MUST match `output_schema` exactly. Downstream actions reference them via `{{trigger.payload.<key>}}`.
