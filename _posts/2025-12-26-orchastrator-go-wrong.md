---
layout: post
title: Orchestrator, What could possibly go wrong
date: 2025-12-27
---


This blog post covers:
Enumerating potential failures
Exploring options for recovering from failures


 we’re going to modify this scenario slightly.
Instead of serving static web pages, we’re going to serve an
API. This API is very simple: it takes a POST request with a
body, and it returns a response with the same body. In
other words, it simply echoes the request in the response.


 this chapter will
reflect on what we’ve built thus far and discuss a number of
failure scenarios, both with our orchestrator and with the
tasks running on it.


### in our new scenario
```
 curl -X POST localhost:7777/ -d '{"Msg":"Hello, world!"}' 
{"Msg":"Hello, world!"}

```


In our new scenario as you can see in the response, we get back the same
body that we sent. To make this scenario simpler to use,
I’ve gone ahead and built a Docker image that we can reuse
throughout the rest of the chapter.
 the number of ways this
API can fail if we run it as a task

## Failure scenarios

Failures at the level of the application
Failures at the level of individual tasks
Failures at the level of the orchestration system

### Application Startup failure

A task can fail to start because the task’s application has a
bug in its startup routine.

What happens, however, if we can’t connect to the
database? Maybe it’s down due to some networking
problem. 


It doesn’t really matter why the database is unavailable.
The fact is that our application depends on the database,
and it needs to do something when it is unavailable

It can attempt to retry connecting to the database.
 Today, most languages have at least one
third-party library that provides the framework to perform
retries using exponential backoff

I could have my application attempt to connect to the database in a
separate goroutine, and until it can connect, maybe my
application serves a 503 response with a helpful message
explaining that the application is in the process of starting
up.


### Application Bugs
A task can also fail after having successfully started up
A user queries our service in a way that triggers our new
code in an unexpected way and crashes the API.

### Task failure due to lack of resource 

A task can fail to start because the worker machine doesn’t
have enough resources (memory, CPU or disk).
One solutions is to have a good scheduler that can allocate tasks to capable workers.

The story for disk, however, is a little more nuanced.
Container images consume disk space.

While there are strategies to minimize size, images still
reside on disk and thus consume space.
 other containers running on a node
may be using disk space for their own data. So it is possible
that an orchestrator could schedule a task onto a worker
node, and then when that worker starts up the task, the
task fails because there isn’t enough disk space to download
the container image.


### Task Failure due to Docker SDK or Daemon

Running tasks can also be affected by problems with the
Docker daemon. For example, if the Docker daemon
crashes, then our task will be terminated. Similarly, if we
stop or restart the Docker daemon, then the daemon will
also stop our task. This behavior is the default for the
Docker daemon. Containers can be kept alive while the
daemon is down using a feature called live restore


### Task Failure Due to machine Crash
The most extreme failure scenario is a worker machine
crashing. In the case of an orchestration system, if a worker
machine crashes, then the Docker daemon will obviously
stop running, along with any tasks running on the machine.

also when, situation where a machine is restarted.
any tasks will be terminated and will
need to be restarted.


### Worker Failures

the worker where
the task runs can also fail. But there is a little more involved
at this level. When we say worker, we need to clarify what
exactly we’re talking about. First, there is the worker
component that we’ve written. Second, there is the machine
where our worker component runs.

Our worker component that we’ve written can crash
due to bugs in our code. It can also crash because the
machine it’s running on crashes or becomes otherwise
unavailable

Unlike
the Docker daemon, our worker going down or restarting
does not terminate running containers.

It does mean that
the manager cannot send new tasks to the worker, and it
means that the manager cannot query the worker to get the
current state of running tasks.
while a failure in our worker is inconvenient, it doesn’t
have an immediate effect on running tasks.


### Manager Failures


 final failure scenario to consider involves the manager
component. Remember, we said the manager serves an
administrative function

, any problems with the manager
will only affect those administrative functions.
, the effect would likely be minimal. Running
tasks would continue to run. Users would not, however, be
able to submit new tasks to the system because the
manager would not be available to receive and take action
on those requests. Again, it would be inconvenient but not
necessarily the end of the world.



## Challenges and Recovery Options
As failures in an orchestration system can occur at multiple
levels and have various degrees of effect.

we are going to define some challenges



### Challenges of application failures

 The only
real automated recovery option here is to perform retries
with exponential backoff or some other mechanism

An orchestrator provides us with
some tools for automated recovery from these kinds of
situations, but if a database is down, continually restarting
the application isn’t going to change the situation.

### Challenges of environmental factors

An orchestration system provides a number of tools for
dealing with non-application-specific failures. We can group
the remaining failure scenarios together and call them
environmental failures:
Failures with Docker
Failures with machines
Failures with an orchestrator’s worker
Failures with an orchestrator’s manager


### Challenges of task-level failures

Docker has a built-in mechanism for restarting containers
when they exit. This mechanism is called restart policies
and can be specified on the command line using the `--restart` flag.

Docker supports four restart policies:
`no`—Do nothing when a container exits (this is the
default).
`on-failure`—Restart the container if it exits with a
nonzero status code.
`always`—Always restart the container, regardless of the
exit code.
`unless-stopped`—Always restart the container,
except if the container was stopped.

The restart policy works well when dealing with individual
containers being run outside of an orchestration system. In
most production situations, we run Docker itself as a
systemd unit. systemd, as the initialization system for most
Linux distributions, can ensure that applications that are
supposed to be running are, in fact, running, especially
after a reboot.

For containers running as part of an orchestration system,
however, using Docker’s restart policies can pose problems.
The main problem is that they muddy the waters around
who is responsible for dealing with failures. Is the Docker
daemon ultimately responsible? Or is the orchestration
system? Moreover, if the Docker daemon is involved in handling failures, this adds complexity to the orchestrator
because it will need to check with the Docker daemon to
see if it’s in the process of restarting a container


For Goat,  we will handle task failures ourselves instead of
relying on Docker’s restart policy. This decision, however,
does raise another question: Should the manager or worker
be responsible for handling failures 

The worker is closest to the task, so it seems natural to
have the worker deal with failures. But the worker is only
aware of its own singular existence. Thus, it can only
attempt to deal with failures in its own context. If it’s
overloaded, it can’t make the decision to send the task to
another worker because it doesn’t know about any other
workers

The manager, though farther away from the actual
mechanisms  knows about all the workers in the
cluster, and it knows about the individual tasks running on
each of those workers. Thus, the manager has more options
for recovering from failures than does an individual worker

### Challenges of worker failures


There are failures in the worker
component itself and failures with the machine where the
worker is running.

 if the worker
fails, there isn’t much consequence to running tasks. In
such a state, however, the worker is in a degraded state.

 we could have the manager attempt to fix
the worker component. How? The obvious thing that comes
to mind is for the manager to consider the worker dead and
move all the tasks to another worker. This is a rather blunt
force tactic, however, and could wreak more havoc. If the
manager simply considers those tasks dead and attempts to
restart them on another worker machine, what happens if
the tasks are still running on the machine where the worker
crashed  blindly considering the worker and all of its
tasks dead, the manager could be putting applications into
an unexpected state. This is particularly true when there is
only a single instance of a task.

where a worker machine has crashed, is
also tricky. How are we defining “crashed”

 Does manager performs some other
operation to verify a worker is up
 an ICMP ping? 
 What if the problem was that the worker machine’s network
card died what if the manager and worker
machine were on different network segments and  manager could not talk
to the worker machine due to router problems?

It’s difficult to determine
whether a machine is down—meaning it has crashed or
been powered off and is otherwise not running any tasks—
in which case the Docker daemon is not running, nor are
any of the tasks under its control.



### Challenges of manager failure
The manager component itself could fail, and the machine
on which the manager component is running could fail.


 if the manager dies, regardless of whether it’s the
manager component itself or the machine where it’s
running, there is no effect on running tasks.



. The only difference is
that the worker won’t receive any new tasks

 sake of simplicity, we have said that we will run only a
single manager. So if it fails, we only need to try to recover
a single instance. If the manager component crashes, we
can restart it. (Ideally, we’d run it using an init system like
Systemd or supervisord.) If its datastore gets corrupted, we
can restore it from a backup

Obviously, the ideal state in regard to the manager would
be to run multiple instances of the manager

When running multiple managers, we have to think about
synchronizing state across all the instances. There might be
a primary instance that is responsible for handling user
requests and acting on them. This instance will also be
responsible for distributing the state of all the system’s
tasks across the other managers. If the primary instance
fails, then another can take over its role. At this point, we
start getting into the realm of consensus and the idea of
how systems agree on the state of the world. This is where
things like the Raft protocol come into play,