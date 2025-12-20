---
layout: post
title: Orchestrator, The Worker API
date: 2025-12-19
---


## Exposing the Worker to the Outside World

Last time, we defined how a worker pulls tasks off its queue and starts or stops them. That was useful, but it only worked **locally**.

In a real orchestration system, the worker will not be operating in isolation. A **manager**, running on a completely different machine, needs a way to interact with it.

This means we need a way to **expose the worker’s core functionality** over the network.

That’s where an API comes in.

---

## API Comes to the Rescue

The worker’s API is intentionally simple. It needs to support only a few basic operations:

* Send a task to the worker (which results in the task being started)
* Get a list of all tasks the worker knows about
* Stop a running task

![Worker API Overview](/images/Screenshot%202025-12-19%20225711.jpg)

To keep things simple and familiar, the worker exposes a **web API**. This lets us:

* Communicate over the network
* Use HTTP as the protocol
* Leverage Go’s standard library and existing tooling

---

## API Building Blocks

A basic web API consists of three main components:

* **Handlers**
  Functions that respond to HTTP requests

* **Routes**
  URL patterns that match incoming requests

* **Router**
  A component that maps routes to handlers

We also need **parameterized routes**, where part of the URL changes per request.

For example:

* `/tasks`
* `/tasks/{taskID}`

Here, `{taskID}` is a variable that represents a specific task.

---

## Worker API Endpoints

The worker exposes the following endpoints:

| Method | Route             | Description                          |
| -----: | ----------------- | ------------------------------------ |
|    GET | `/tasks`          | Get a list of all tasks              |
|   POST | `/tasks`          | Create (start) a task                |
| DELETE | `/tasks/{taskID}` | Stop the task identified by `taskID` |

The API itself is built using:

* Go’s `net/http` package
* The `chi` router
* Our own `Worker` implementation

---

## The API Struct

The API is defined in `worker/api.go`.

Its job is to **wrap the worker** and expose the worker’s functionality over HTTP.

```go
type Api struct {
    Address string
    Port    int
    Worker  *Worker
    Router  *chi.Mux
}
```

Field breakdown:

* **Address / Port**
  Where the API server will listen

* **Worker**
  A pointer to the worker instance whose functionality we are exposing

* **Router**
  A `chi.Mux` router used to define routes and connect them to handlers

---

## Handling Requests

A handler is simply a function that knows how to respond to an HTTP request.

For the worker API, we define three handler methods on the `Api` struct:

```go
StartTaskHandler(w http.ResponseWriter, r *http.Request)
GetTasksHandler(w http.ResponseWriter, r *http.Request)
StopTaskHandler(w http.ResponseWriter, r *http.Request)
```

Each handler is responsible for translating HTTP requests into worker actions.

---

## StartTaskHandler

At a high level, `StartTaskHandler`:

1. Reads the request body from `r.Body`
2. Decodes the JSON payload into a `task.TaskEvent`
3. Adds the task event to the worker’s queue
4. Logs what happened
5. Writes an appropriate HTTP status code to the response

To make this safer, we use:

```go
decoder.DisallowUnknownFields()
```

This forces the JSON decoder to fail if the request contains fields that don’t exist in `task.TaskEvent`. It helps catch malformed requests early.

---

## GetTasksHandler

This handler looks simple, but there’s quite a bit happening:

1. Set the `Content-Type` header to `application/json`
2. Set an HTTP status code
3. Create a `json.Encoder`
4. Fetch all tasks using `worker.GetTasks()`
5. Encode the task list as JSON and write it to the response

This endpoint gives the manager visibility into what the worker is currently running.

---

## StopTaskHandler

Stopping a task is done via:

```text
DELETE /tasks/{taskID}
```

The steps are:

1. Extract `taskID` from the URL using `chi.URLParam`
2. Convert it from `string` to `uuid.UUID` using `uuid.Parse`
3. Check if the worker knows about this task

   * If not, return `404 Not Found`
4. If it exists:

   * Change the task state to `Completed`
   * Add it to the worker’s queue

One subtle but important detail here is **copying the task**.

The worker’s:

* **DB** represents the *current* state
* **Queue** represents the *desired* state

Because of this, we dereference the task from the DB and enqueue a copy with the updated state.

---

## Serving the API (Routes)

The `Api` struct already contains everything we need:

* A `Worker` to perform operations
* A `Router` to route requests

To actually serve the API, we make two additions to `api.go`.

---

### initRouter()

This method initializes routing:

1. Create a router using `chi.NewRouter()`
2. Define routes for:

   * `GET /tasks`
   * `POST /tasks`
   * `DELETE /tasks/{taskID}`

This connects each route to its corresponding handler.

---

### Start()

The `Start()` method brings everything to life:

1. Call `initRouter()`
2. Start an HTTP server using `http.ListenAndServe`

The server listens on the configured address and port and begins accepting requests.

---

## One More Worker Method: GetTasks

For `GetTasksHandler` to work, the worker needs a helper method:

```text
GetTasks()
```

This method simply returns all tasks stored in the worker’s DB.

It lives in `worker.go` and keeps API logic separate from worker internals.

---

## Letting the API Serve Us (main.go)

We continue using `main.go` as our entry point.

The `main()` function does the following:

1. Create a worker `w` with a queue and DB
2. Create an API `api`, using host and port values from the environment
3. Start the system

Two important calls make everything work:

---

### Running Tasks Concurrently

```go
go runTasks(&w)
```

The `go` keyword starts `runTasks` in a separate goroutine.

If you’re familiar with threads in other languages, this is roughly the same idea.

---

### Starting the API

```go
api.Start()
```

This starts the HTTP server and blocks the main thread.

---

## runTasks Function

The `runTasks` function is intentionally simple:

* It runs in an infinite loop
* Checks the worker’s queue
* Calls `RunTask()` when tasks are available
* Sleeps for 10 seconds between iterations

The sleep makes logs easier to read while experimenting.

---

## Separation of Concerns

One important design decision here is that:

> The API does **not** start or stop tasks directly.

Instead:

* The API **enqueues desired state**
* The worker **reconciles state and performs actions**

This clean separation makes the system easier to reason about and evolve over time.

At this point, we have a worker that can be controlled remotely — a major step toward a real orchestration system.
