---
layout: post
title: Orchestrator, The Manager II — API
date: 2025-12-24
---


## Why the Manager Needs an API

Up until now, we’ve mostly focused on the internal mechanics of the manager. The next logical step is to **expose the manager to users**.

Here, “users” means **application developers** who want to run their workloads on our orchestration system. They shouldn’t care about workers, containers, or scheduling details. They just want a clean interface to interact with the system.

So, we build an **API for the manager**.

This API wraps the manager’s core functionality and exposes it in a user-friendly way.

---

## What the Manager API Should Do

The manager API needs to support only a small set of operations:

* Send a task to the manager
* Get a list of all tasks in the system
* Stop a task

Just like the worker API, the manager API is built using:

* **Handlers**
* **Routes**
* **A router (mux)**

---

## Zooming Out: The Big Picture

Before diving into code-level details, it helps to zoom out and look at the overall architecture.

The **manager** and **worker** are abstractions that free developers from having to think about the underlying infrastructure—whether it’s physical machines or virtual ones.

Each of them follows the same basic pattern:

* An **API server**
* A **core component** (manager logic or worker logic)

![Image](https://fabric.inc/wp-content/uploads/2023/05/api-orchestration-diagram.gif)

![Image](https://pinggy.io/assets/control_plane.png)

This symmetry is intentional and makes the system easier to reason about.

---

## Manager API Routes

Now let’s zoom back in and look at the manager’s API itself.

The routes are intentionally very similar to the worker’s API:

* **GET `/tasks`**
  Returns a list of all tasks in the system

* **POST `/tasks`**
  Creates (starts) a new task

* **DELETE `/tasks/{taskID}`**
  Stops a task identified by `taskID`

All data sent to the API must be **JSON-encoded**, and all responses are returned as **JSON** as well.

---

## Manager API Summary

| Method | Route             | Description                         | Request Body          | Response Body | Status |
| -----: | ----------------- | ----------------------------------- | --------------------- | ------------- | ------ |
|    GET | `/tasks`          | Gets a list of all tasks            | None                  | List of tasks | 200    |
|   POST | `/tasks`          | Creates a task                      | JSON `task.TaskEvent` | None          | 201    |
| DELETE | `/tasks/{taskID}` | Stops the task identified by taskID | None                  | None          | 204    |

---

## The API Struct

Just like the worker, the manager has its own `Api` struct.

The structure is almost identical to the worker’s API. The only real difference is this:

* The `Worker` field is replaced with a `Manager` field

In other words, the API now wraps a **manager instance** instead of a worker instance.

---

## Handling Requests

The manager API defines three handler methods:

```text
StartTaskHandler(w http.ResponseWriter, r *http.Request)
GetTasksHandler(w http.ResponseWriter, r *http.Request)
StopTaskHandler(w http.ResponseWriter, r *http.Request)
```

These handlers translate HTTP requests into manager actions.

---

## StartTaskHandler

This handler expects a JSON-encoded request body.

High-level flow:

1. Decode the request body into a `task.TaskEvent`
2. Check for decoding errors
3. Add the task event to the manager’s queue using `AddTask`
4. Return a `201 Created` status

Notice that the handler **does not start tasks directly**. It simply enqueues intent.

---

## GetTasksHandler

To support this handler cleanly, we add a helper method to the manager:

### `GetTasks()`

This method:

1. Creates a slice of `*task.Task`
2. Iterates over the manager’s `TaskDb`
3. Appends each task to the slice
4. Returns the slice

The handler then:

* Sets the response headers
* Encodes the task list as JSON
* Writes it to the response

---

## StopTaskHandler

This handler mirrors the worker’s stop handler closely.

Steps:

1. Extract `taskID` from the request path
2. Check if the task exists in the manager’s database
3. Create a new `task.TaskEvent` wrapping the task
4. Make a copy of the task to avoid mutating existing state
5. Add the task event to the manager’s queue
6. Return a `204 No Content` response

Again, the handler is only expressing **desired state**, not performing the stop itself.

---

## Separation of Concerns (Again)

Just like with the worker API, the manager’s handlers:

* Do **not** start or stop tasks directly
* Only enqueue intent

The actual execution happens elsewhere. This keeps API logic clean and focused.

---

## Serving the Manager API

Serving the manager API looks almost identical to serving the worker API.

We reuse the same `initRouter` pattern:

* Create a router
* Define routes
* Attach handlers

Then we start the server using `http.ListenAndServe`.

Because the routes are the same as the worker’s, very little new code is required here.

---

## Refactoring for Cleaner Code

As the system grows, a few refactorings make life easier.

### Moving `runTasks` into the Worker

Previously, `runTasks` lived in `main.go`. This isn’t ideal.

We move it into the `Worker` itself so all worker behavior is encapsulated.

Changes made:

* Rename `RunTask` → `runTask`
* Rename `runTasks` → `RunTasks`
* Update `RunTasks` to call `runTask`

This keeps `main.go` cleaner and more focused on wiring components together.

---

### Manager Loop Refactoring

We also refactor the manager slightly:

* Rename `UpdateTasks` → `updateTasks`
* Add a new public `UpdateTasks` method that runs an infinite loop
* Add a `ProcessTasks` method that repeatedly calls `SendWork`

These methods mirror the worker’s looping behavior and make the manager easier to run.

---

## Putting It All Together (main.go)

Finally, we wire everything together in `main.go`.

High-level flow:

1. Read host and port values from the environment
2. Start the worker API
3. Start the worker’s `RunTasks` loop
4. Create a list of workers (`<host>:<port>`)
5. Create a manager instance
6. Create the manager API (`mapi`)
7. Start manager background loops:

   * `ProcessTasks`
   * `UpdateTasks`
8. Start the manager API server

The manager loops run in separate goroutines, while the main goroutine blocks on the API server.

---



With this in place, users finally have a **single entry point** into the system.

They talk only to the manager.
The manager coordinates workers.
Workers execute tasks.

The architecture is still simple, but it’s now structured in a way that closely resembles real-world orchestrators — just without the complexity (yet).
