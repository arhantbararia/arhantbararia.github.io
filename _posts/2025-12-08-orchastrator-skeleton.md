---
layout: post
title: Orchestrator Skeleton
date: 2025-12-08
---


## Translating a Model into Code (Sort of)

I like to translate a system model into **skeleton code** early on.
This helps me think about the implementation in a more concrete way without getting too deep into details too soon.

At a high level, an orchestrator naturally breaks down into a few obvious components:

* **Manager**
* **Worker**
* **Scheduler**

At the foundation of all of these components, however, is the **Task**.
Everything in the system exists to create, move, run, stop, or observe tasks.

Below is a basic skeleton I found online that inspired this structure:

![Basic Skeleton](/images/Screenshot%202025-12-09%20014025.jpg)

---

## The Task Skeleton

### Task States

A task in the system moves through a well-defined lifecycle. I model this explicitly using states.

* **Pending**
  The task has been enqueued but not yet scheduled.

* **Scheduled**
  The system has decided which machine should run the task, but the task is still in the process of being sent or started.

* **Running**
  The selected machine has successfully started the task.

* **Completed**
  The task finished successfully or was stopped intentionally by a user.

* **Failed**
  The task crashed or stopped working as expected at any point.

![Task State Diagram](/images/Screenshot%202025-12-09%20014218.jpg)

---

### Task Identity

Each task needs a stable identifier.
For this, I use **UUIDs**.

```go
type Task struct {
    ID    uuid.UUID
    Name  string
    State State
}
```

The `State` field refers to the task state enum defined earlier.

---

### Task Configuration

As the orchestrator evolves, the task structure needs more metadata:

* **Image** – Docker image to run
* **CPU, Memory, Disk** – resource requirements
* **ExposedPorts / PortBindings** – Docker networking
* **RestartPolicy** – restart behavior
* **StartTime / FinishTime** – lifecycle tracking

A more complete task structure looks like this:

```go
type Task struct {
    ID            uuid.UUID
    ContainerID   string
    Name          string
    State         State
    Image         string
    CPU           float64
    Memory        int64
    Disk          int64
    ExposedPorts  nat.PortSet
    PortBindings  map[string]string
    RestartPolicy string
    StartTime     time.Time
    FinishTime    time.Time
}
```

One important open question here is:
**How does a user tell the system to stop a task?**
This becomes clearer once we introduce events.

---

## Project Structure

At this stage, the repository layout looks something like this:

```text
.
├── main.go
├── manager
│   └── manager.go
├── node
│   └── node.go
├── scheduler
│   └── scheduler.go
├── task
│   └── task.go
└── worker
    └── worker.go
```

Each major concept gets its own package.

---

## TaskEvent

Tasks don’t change state by themselves.
State transitions are triggered via **events**.

A `TaskEvent` represents a request to move a task from one state to another.

* **State** – the target state (e.g., Running → Completed)
* **Timestamp** – when the event was requested
* **Task** – the task being affected

```go
type TaskEvent struct {
    ID        uuid.UUID
    State     State
    Timestamp time.Time
    Task      Task
}
```

This is an internal object used by the system to drive task state transitions.

---

## The Worker Skeleton

The worker is the layer that actually runs tasks.

Responsibilities:

1. Run tasks as Docker containers
2. Accept tasks from the manager
3. Provide statistics for scheduling decisions
4. Track tasks and their states locally

To do this, the worker maintains:

* **Db** – an in-memory map of task IDs to tasks
* **Queue** – tasks sent from the manager
* **TaskCount** – number of tasks currently running

```go
type Worker struct {
    Name      string
    Queue     queue.Queue
    Db        map[uuid.UUID]*task.Task
    TaskCount int
}
```

### Worker Methods

Some key methods that actually do the work:

* **RunTask**
  Inspects the task state and decides whether to start or stop it.

* **StartTask**
  Starts a task container.

* **StopTask**
  Stops a running task.

* **CollectStats**
  Periodically gathers resource usage and task metrics.

---

## The Manager Skeleton

The manager coordinates the entire system.

Responsibilities:

1. Accept user requests to start or stop tasks
2. Schedule tasks onto workers
3. Track task state and task-to-worker assignments

Key fields:

* **pending** – a FIFO queue of tasks waiting to be scheduled
* **tasks** – in-memory database of tasks
* **events** – in-memory database of task events
* **workers** – list of worker names in the cluster

To track assignments:

* **WorkerTaskMap** – maps worker name → task IDs
* **TaskWorkerMap** – maps task ID → worker name

The manager also needs scheduling logic.

### Manager Methods

* **selectWorker**
  Chooses the best worker for a task based on resource requirements.

* **UpdateTasks**
  Refreshes task states and typically triggers `CollectStats` on workers.

* **SendWork**
  Sends tasks to the selected worker.

---

## The Scheduler Skeleton

Scheduling logic deserves its own abstraction.

Responsibilities:

1. Select candidate workers for a task
2. Score those workers
3. Pick the best one

Initially, a **round-robin** scheduler is enough:

* Track the last worker that received a task
* Assign the next task to the next worker in the list

To allow flexibility, scheduling is defined via an interface:

```go
type Scheduler interface {
    SelectCandidateNodes()
    Score()
    Pick()
}
```

This makes it easy to experiment with different scheduling algorithms later.

---

## The Node Skeleton

A worker has a physical aspect: it runs on a machine.
That physical machine is modeled as a **Node**.

A node represents any machine in the cluster.

Key fields:

* **Name**
* **IP** – used by the manager to communicate
* **Memory / Disk** – total available resources
* **MemoryAllocated / DiskAllocated** – resources currently in use
* **TaskCount** – number of tasks running on the node

The node exists primarily to expose machine-level stats for scheduling decisions.

---

## The Main Runner

Finally, `main.go` ties everything together.

At this early stage, it does very basic wiring:

* Create a `Task`
* Create a `TaskEvent`
* Print both
* Create a `Worker`
* Print the worker
* Call worker methods
* Create a `Manager`
* Call manager methods
* Create a `Node`
* Print the node

Nothing fancy yet — just enough structure to start reasoning about how all the pieces fit together.

---

This skeleton doesn’t *do* much yet, but it gives me a solid mental model and a concrete starting point to iterate on.
