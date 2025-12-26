---
layout: post
title: Orchestrator, What Could Possibly Go Wrong
date: 2025-12-27
---

As Edward Murphy Jr. famously said: "Anything that can go wrong will go wrong." 

This post is a bit of a reality check.

So far, we’ve been building pieces of an orchestrator and wiring them together. Things mostly work in the happy path. But real systems don’t live in the happy path for long.

This post focuses on two things:

* Enumerating potential failure scenarios
* Exploring options for recovery

To make the discussion more concrete, we’ll slightly change our running example.

---

## A Slightly Different Scenario

Instead of serving static web pages, let’s imagine we’re running a very simple API as a task.

The API does exactly one thing:

* It accepts a POST request
* It responds with the same body (an echo server)

Example:

```bash
curl -X POST localhost:7777/ -d '{"Msg":"Hello, world!"}'
{"Msg":"Hello, world!"}
```

As you can see, the response body is identical to the request body.

To keep things simple, I’ve packaged this API as a Docker image so we can reuse it throughout this discussion. Even with something this basic, there are *many* ways things can go wrong once we run it as a task inside an orchestrator.

---

## Failure Scenarios

Failures can happen at multiple levels in the system. Broadly, we can group them into three categories:

* Failures at the **application** level
* Failures at the **task** level
* Failures at the **orchestration system** level

Let’s walk through each.

---

## Application Startup Failures

A task can fail to start simply because the application itself has a bug.

A common example is a dependency failure. Imagine our API depends on a database. What happens if the database is unavailable?

* Maybe the database is down
* Maybe there’s a networking issue
* Maybe credentials are wrong

The exact reason doesn’t matter. What matters is that the application depends on something external and must react when that dependency isn’t available.

Common strategies include:

* Retrying the connection
* Using exponential backoff
* Serving a `503 Service Unavailable` response while the app is starting up

For example, the application could try to connect to the database in a separate goroutine and, until it succeeds, respond with a helpful message explaining that it’s still starting.

---

## Application Bugs

Even if a task starts successfully, it can still fail later.

A user might send a request that triggers an edge case in the code, causing the API to crash. These kinds of failures are often the hardest to predict because they depend on real-world usage patterns.

---

## Task Failure Due to Lack of Resources

A task can also fail because the worker machine doesn’t have enough resources:

* Not enough memory
* Not enough CPU
* Not enough disk

For CPU and memory, a decent scheduler can help by placing tasks only on workers that have sufficient capacity.

Disk is a bit trickier.

Container images take up disk space. Even if an image is small, it still has to be downloaded and stored locally. On top of that, other containers on the same node might be using disk space for their own data.

This means an orchestrator might schedule a task onto a worker that *looks* capable, only for the task to fail when Docker tries to pull the image and runs out of disk space.

---

## Task Failure Due to Docker SDK or Daemon Issues

Tasks can also fail due to problems with Docker itself.

If the Docker daemon crashes, any running containers will be terminated. Restarting or stopping the daemon also stops containers by default.

Docker does offer a feature called **live restore**, which allows containers to keep running while the daemon is down, but this isn’t always enabled.

---

## Task Failure Due to Machine Crash

One of the most extreme scenarios is a worker machine crashing.

If a machine crashes or is restarted:

* The Docker daemon stops
* All running containers stop
* All tasks running on that machine are terminated

Those tasks will need to be restarted somewhere else.

---

## Worker Failures

When we talk about worker failures, we need to be precise.

There are two things that can fail:

1. The **worker component** we wrote
2. The **machine** the worker is running on

Our worker component can crash due to bugs in our code. The machine itself can crash or become unreachable.

Unlike the Docker daemon, if the worker component crashes, running containers are *not* automatically terminated. That’s good news.

However, a failed worker means:

* The manager can’t send new tasks to it
* The manager can’t query it for task state

So while a worker failure doesn’t immediately affect running tasks, it does leave the system in a degraded state.

---

## Manager Failures

Finally, we have to consider failures of the manager.

The manager handles administrative concerns. If it fails:

* Running tasks continue to run
* Workers keep executing what they were already doing
* Users cannot submit new tasks

This is inconvenient, but not catastrophic.

Since we’ve assumed a single manager for simplicity, recovery is relatively straightforward:

* Restart the manager if it crashes
* Run it under an init system like `systemd` or `supervisord`
* Restore its datastore from backup if needed

---

## Challenges and Recovery Options

Failures happen at many levels and have very different effects. Let’s group the recovery challenges.

---

## Challenges of Application Failures

For application-level failures, the orchestrator has limited options.

Retries with exponential backoff are often the only automated recovery mechanism. But if a dependency like a database is down, restarting the application over and over won’t magically fix the problem.

---

## Challenges of Environmental Failures

Environmental failures include:

* Docker daemon failures
* Machine crashes
* Worker failures
* Manager failures

Orchestration systems exist largely to help deal with these kinds of problems, but none of them are trivial.

---

## Challenges of Task-Level Failures

Docker provides **restart policies** for containers:

* `no` : Do nothing (default)
* `on-failure` : Restart on nonzero exit code
* `always` : Always restart
* `unless-stopped` : Restart unless explicitly stopped

These work well when running standalone containers. In production, Docker itself is often managed by `systemd`, which ensures it restarts after reboots.

In an orchestration system, though, restart policies introduce ambiguity.

Who is responsible for recovery?

* Docker?
* The worker?
* The orchestrator as a whole?

If Docker Daemon is restarting containers on its own, the orchestrator has to ask Daemon what it’s doing, which adds complexity.

For GOAT, the decision is to **handle task failures ourselves**, rather than relying on Docker’s restart policies.

This raises another question: should failures be handled by the worker or the manager?

* The worker is closest to the task, but it only knows about itself.
* The manager knows about *all* workers and tasks in the cluster.

Because of that global view, the manager has more options for recovery.

---

## Challenges of Worker Failures

Worker failures are particularly tricky.

If the worker component fails but the machine is still running, tasks may still be executing. Blindly assuming that all tasks are dead and restarting them elsewhere could cause duplicate execution or inconsistent state—especially if there’s only a single instance of a task.

Detecting whether a worker machine has truly crashed is also hard.

* Do we ping it?
* What if the network is partitioned?
* What if routing is broken?

Determining whether a machine is actually down or just unreachable is one of the hardest problems in distributed systems.

---

## Challenges of Manager Failure

If the manager fails, running tasks continue unaffected. The main impact is administrative:

* No new tasks can be submitted
* No scheduling decisions can be made

With a single manager, recovery is manageable. With multiple managers, things get much more complex.

At that point, managers need to agree on system state. You may have a primary manager that handles user requests and replicates state to others. If the primary fails, another manager takes over.

This is where we enter the world of **consensus**, **leader election**, and protocols like **Raft**. We are not going to go into that.

---

 
 As we've seen, building a robust orchestrator means constantly thinking about what can go wrong. From application bugs to machine crashes, the path to a resilient system is paved with failure scenarios.
 
 While we won't delve into complex distributed systems concepts like Raft for now, understanding these potential pitfalls is the first step toward designing a system that can gracefully recover.
 
 
