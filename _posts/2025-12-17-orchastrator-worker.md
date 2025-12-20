---
layout: post
title: Orchestrator, The Worker
date: 2025-12-17
---


## The Worker: Where the Work Actually Happens

In an orchestration system, the **worker** is responsible for actually *doing* the work.

At a very high level, the worker’s job is simple:

* Start tasks
* Stop tasks

In practice, this makes the worker one of the most important components in the system.

In this post, I’ll focus on fleshing out the **Worker skeleton** we sketched earlier, using the **Task** and **Docker** abstractions we already built.

---

## Why Do We Need Workers?

Consider a simple example: a web server that serves static pages.

In many cases, running a **single instance** of this web server on one machine is enough. But as traffic grows, a single machine can become a bottleneck:

* CPU limits
* Memory pressure
* Network saturation

Running **multiple instances** of the same web server across different machines helps solve these problems. This is where the worker component becomes useful.

The worker allows us to scale applications horizontally by running multiple instances of the same task across machines.

![Worker Concept](/images/Screenshot%202025-12-17%20195209.jpg)
---

## A Small Terminology Note

The term **Worker** does double duty:

1. It represents a **physical or virtual machine**
2. It also represents the **worker component** of the orchestration system running on that machine

This dual meaning is important to keep in mind as the design evolves.


With multiple workers available, we’re far less likely to run into resource availability issues.

---

## Worker Internals

The worker itself is composed of several smaller pieces:

1. **API** – receives work
2. **Runtime** – runs tasks (Docker, in our case)
3. **Task Queue** – incoming desired state changes
4. **Task Database (DB)** – tracks task state
5. **Metrics** – reports stats to the manager

![Worker Components](/images/Screenshot%202025-12-17%20195416.jpg)

---

## Tasks and Containers

From our task structure, we know that a task performs its work by running as a **Docker container**.

This gives us a clean, simple relationship:

> **One task = one container**

Two key structs are involved:

* **Task struct**
  Used by the worker to manage state and queues

* **Docker struct**
  Used to start and stop tasks as containers

The worker doesn’t need to know *how* Docker works internally — it just calls `Run` and `Stop`.

---

## The Worker Struct

We previously defined the worker like this:

```go
type Worker struct {
    Name      string
    Queue     queue.Queue
    Db        map[uuid.UUID]*task.Task
    TaskCount int
}
```

A few important points:

* **Queue**
  This is composition in Go. The worker embeds another type instead of reimplementing queue logic.

* **Db**
  An in-memory map that tracks tasks and their state.
  Later, this can be replaced with a persistent datastore.

* **TaskCount**
  A simple counter tracking how many tasks the worker is responsible for.

---

## Core Worker Methods

Earlier, we defined three methods but left them incomplete:

1. `RunTask`
2. `StartTask`
3. `StopTask`

Now we’ll fill them in.

---

## Implementing `StopTask`

The `StopTask` method has one job: **stop a running task**.

High-level steps:

1. Create a `Docker` object to talk to the Docker daemon
2. Call the `Stop()` method on the Docker object
3. Check for errors
4. Update the task’s `FinishTime`
5. Save the updated task in the worker’s `Db`
6. Print a useful message and return the result

This closely mirrors running `docker stop` followed by `docker rm`.

---

## Implementing `StartTask`

The `StartTask` method does the opposite.

Steps:

1. Update the task’s `StartTime`
2. Create a `Docker` object
3. Call the `Run()` method
4. Check for errors
5. Store the container ID in `t.Runtime.ContainerId`
6. Save the updated task in the worker’s `Db`
7. Return the result

The key idea here is **encapsulation**.

By putting all Docker-related logic inside the Docker object, the worker doesn’t need to know anything about Docker internals.

---

## Implementing `AddTask`

This method is intentionally simple.

Its only responsibility is to add a task `t` to the worker’s queue.

The queue represents the **desired state** of a task.

---

## Managing Tasks with `RunTask`

The `RunTask` method is where things get interesting.

Its job is to:

* Look at a task’s current state
* Compare it with the desired state
* Decide whether to start or stop the task

There are two scenarios:

1. **First submission**
   The task is not in the worker’s DB yet.

2. **Subsequent submission**
   The task already exists, and the new task represents a desired state change.

In our naive implementation:

* The **Queue** represents desired state
* The **Db** represents current state

If a task is in the queue but not in the DB, the worker assumes it needs to be started.

---

## Validating State Transitions

Not all state transitions should be allowed.

For example:

* Can a task move from `Running` back to `Scheduled`?
* Should a `Failed` task be allowed to transition to `Scheduled`?

Real orchestrators like **Borg**, **Kubernetes**, and **Nomad** solve this using state machines.

We’ll do something simpler: **hard-code valid transitions**.

```go
var stateTransitionMap = map[State][]State{
    Pending:   []State{Scheduled},
    Scheduled: []State{Scheduled, Running, Failed},
    Running:   []State{Running, Completed, Failed},
    Completed: []State{},
    Failed:    []State{},
}
```

Each key is the current state, and the value is a list of allowed next states.

---

## Helper Functions

To support this, we need two helpers:

* **Contains**
  Checks whether a state exists in a slice of states

* **ValidStateTransition**
  Wraps `Contains` to check if a transition is allowed

These helpers keep the `RunTask` logic clean and readable.

---

## Implementing `RunTask`

The `RunTask` method follows this flow:

1. Dequeue a task
2. Convert it from `interface{}` to `task.Task`
3. Retrieve the existing task from the DB
4. Validate the state transition
5. If desired state is `Scheduled`, call `StartTask`
6. If desired state is `Completed`, call `StopTask`
7. Otherwise, return an error

Because the queue stores values as `interface{}`, we need a type assertion:

```go
t := w.Queue.Dequeue()
taskQueued := t.(task.Task)
```

This tells Go what concrete type we expect.

---

## Putting It All Together (main.go)

The `main.go` file ties everything together.

Flow:

1. Create a worker `w` with a queue and DB
2. Create a task `t` with:

   * State: `Scheduled`
   * Image: `strm/helloworld-http`
3. Call `w.AddTask(t)`
4. Call `w.RunTask()`

At this point, the container is running.

After sleeping for about 30 seconds:

1. Update the task state to `Completed`
2. Call `AddTask` again with the same task
3. Call `RunTask` again

This time, the worker sees the state change and stops the running container.

---

## Final Thoughts

This implementation is intentionally naive, but it demonstrates the core idea:

* The **queue** represents desired state
* The **DB** represents current state
* The **worker** reconciles the two

Even with this simple setup, we already have the foundations of a real orchestration system. From here, we can improve durability, scheduling, and fault tolerance step by step.
