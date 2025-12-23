```md
---
layout: post
title: Orchestrator, The Manager
date: 2025-12-23
---
```

## Why Do We Need a Manager?

Even though our system has **multiple workers**, we should *not* ask users to submit their tasks directly to a worker.

Why?

Because that would put an unnecessary burden on users. They would need to know:

* How many workers exist
* Which workers are already busy
* Which worker is best suited for their task

Instead of pushing this complexity onto users, we **centralize all of this logic inside a single component**: the **Manager**.

---

## Single Manager Design

Our orchestrator will have **one manager**.

This is a deliberate design choice. It keeps the system simpler and reduces the number of problems we need to think about at this stage (leader election, consensus, split brain, etc.).

The manager allows us to clearly separate:

* **Administrative concerns** → Manager
* **Execution concerns** → Worker

This design principle is commonly known as **separation of concerns**.

---

## Manager Responsibilities

Administrative responsibilities in an orchestration system include:

* Handling requests from users
* Scheduling tasks onto appropriate workers
* Keeping track of task state
* Keeping track of worker state
* Restarting failed tasks

![The manager is responsible for administrative tasks](/images/Screenshot%202025-12-23%20225805.jpg)

---

## Control Plane vs Data Plane

Another useful way to think about this separation is:

* **Control Plane** → Manager
* **Data Plane** → Worker

This terminology comes from networking.

* The **control plane** decides *what should happen*
* The **data plane** actually *does the work*

In our system:

* The manager decides *where* a task should run
* The worker actually *runs* the task

---

## Components of the Manager

The manager is composed of several internal subcomponents.

### TaskDB

Like the worker, the manager maintains a task database.
The difference is:

* **Worker TaskDB** → only tasks running on that worker
* **Manager TaskDB** → *all tasks in the orchestration system*

---

### EventDB

The manager also maintains an **EventDB**.

This database stores task events rather than tasks themselves.
It helps separate **metadata** from **task state**.

Examples of metadata:

* When a user submitted a task
* When a task transition was requested

---

### Workers

The manager keeps a list of all workers in the cluster.

Each worker is represented as a string, typically in the form:

```
<hostname>:<port>
```

This corresponds to the address where the worker’s API is running.

---

### Task Queue

Just like the worker, the manager also has a **queue**.

* The queue represents *pending task events*
* Tasks enter the system here before being scheduled

---

### API

Finally, the manager exposes an API that users interact with.

Users never talk to workers directly — they only talk to the manager.

---

## The Manager Struct

Putting all of this together, the manager struct looks like this:

```go
type Manager struct {
    Pending       queue.Queue
    TaskDb        map[uuid.UUID]*task.Task
    EventDb       map[uuid.UUID]*task.TaskEvent
    Workers       []string
    WorkerTaskMap map[string][]uuid.UUID
    TaskWorkerMap map[uuid.UUID]string
}
```

A few important mappings here:

* **WorkerTaskMap**
  Worker → list of tasks running on that worker

* **TaskWorkerMap**
  Task → worker running it

These maps make it easy to answer questions like:

* “Which worker is running this task?”
* “Which tasks are running on this worker?”

---

## Core Manager Methods

We implement the manager in stages. The main methods are:

1. `SelectWorker`
2. `SendWork`
3. `UpdateTasks`

---

## `SelectWorker`: A Naive Scheduler

At this stage, the manager *is* the scheduler.

The `SelectWorker` method chooses a worker from the `Workers` list.

We start with a **round-robin** algorithm:

* Pick the next worker in the list
* Move forward one index each time
* Wrap around when we reach the end

To support this, we add one more field to the manager:

```go
LastWorker int
```

This field stores the index of the last worker that was selected.

The algorithm works like this:

* Select `Workers[LastWorker]`
* Increment `LastWorker`
* If we reach the end, reset to `0`

Simple, but good enough to start.

---

## `SendWork`: The Workhorse Method

`SendWork` is where most of the manager’s logic lives.

Its responsibilities are:

1. Check if there are task events in the pending queue
2. Select a worker
3. Pull a task event from the queue
4. Set the task state to `Scheduled`
5. Update internal bookkeeping maps
6. JSON-encode the task event
7. Send the task to the selected worker
8. Check the worker’s response

The method begins by checking:

* Is `Pending` queue non-empty?

If yes, we proceed.

---

### Sending Work to a Worker

Once a worker is selected, the manager builds the worker’s API URL using:

```
<hostname>:<port>
```

Then it sends the task event using an HTTP `POST` request via Go’s `net/http` package.

At this point:

* The manager has *delegated* the task
* The worker is responsible for execution

---

## `UpdateTasks`: Syncing State from Workers

A successful HTTP response from a worker only means:

> “The worker accepted the task.”

It does **not** mean:

* The task started successfully
* The task is still running
* The task didn’t fail immediately

Because of this, the manager must **periodically reconcile state**.

That’s the job of `UpdateTasks`.

---

### How `UpdateTasks` Works

For each worker in the cluster:

1. Send a `GET /tasks` request to the worker
2. Decode the response (list of tasks)
3. For each task:

   * Compare worker state with manager state
   * Update manager state if they differ
   * Update `StartTime`
   * Update `FinishTime`
   * Update `ContainerID`

In this design, **workers are the source of truth** for task state.

The manager simply reflects what workers report.

---

## Additional Manager Methods

### `AddTask`

This method should feel familiar.

Just like the worker’s `AddTask`, it:

* Adds a task event to the manager’s queue

This is how new work enters the system.

---

## Creating a Manager

Finally, we create a helper constructor:

### `New`

This function:

* Accepts a list of workers
* Initializes:

  * TaskDB
  * EventDB
  * WorkerTaskMap
  * TaskWorkerMap
* Returns a pointer to a fully initialized manager

This keeps setup logic clean and centralized.

---
