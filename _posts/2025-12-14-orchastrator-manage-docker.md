---
layout: post
title: Orchestrator, Task & Docker
date: 2025-12-14
---


## Docker and the SDK

I like to think of containers like a **kitchen** for a task.

Just like a kitchen provides utensils, gas, and space to cook, a container provides everything a task needs to run:

* CPU cycles
* Memory
* Networking

Containers allow developers to package their code along with all its dependencies and ship it reliably to production. This makes them a very natural fit for an orchestration system.

To interact with Docker programmatically, we’ll use **Docker’s Go SDK**. The SDK hides the low-level HTTP details and lets us focus on what we actually want to do: **create, run, and stop containers**.

We’ll rely mainly on the following SDK methods:

* **NewClientWithOpts** – creates a Docker client
* **ImagePull** – pulls an image from a registry
* **ContainerCreate** – creates a container
* **ContainerStart** – starts a container
* **ContainerStop** – stops a running container
* **ContainerRemove** – removes a container from the host

Official documentation:
[https://pkg.go.dev/github.com/docker/docker/client](https://pkg.go.dev/github.com/docker/docker/client)

---

## Task Configuration

### Config Struct

The `Config` struct encapsulates everything needed to describe *how* a task should run.

Key fields:

* **Image**
  The Docker image to run.
  Example: `postgres` or a specific version like `postgres:13`

* **Memory / Disk**
  These serve two purposes:

  1. Help the scheduler choose a suitable node
  2. Define container-level resource constraints

* **Env**
  Environment variables passed into the container

* **RestartPolicy**
  One of:

  * `always`
  * `unless-stopped`
  * `on-failure`

Restart policies are one of the core mechanisms that provide resilience in the orchestrator.

* `always` – restart whenever the container stops
* `unless-stopped` – restart unless a user explicitly stops it
* `on-failure` – restart only if the container exits with a non-zero code

---

## Starting and Stopping Tasks

The **worker** is responsible for actually running tasks, and Docker is the engine that makes this possible.

### Docker Struct

The `Docker` struct acts as a thin wrapper around the Docker SDK.

* **Client** – Docker SDK client
* **Config** – task configuration

This keeps Docker-specific logic isolated from the rest of the worker code.

---

### DockerResult Struct

For convenience, we define a `DockerResult` struct to standardize responses from Docker operations.

Fields include:

* **Action** – what was attempted (`start`, `stop`, etc.)
* **ContainerID** – ID of the container
* **Result** – human-readable output
* **Error** – any error that occurred

This makes it easier for callers to reason about what happened.

---

## Run Method

The `Run` method is responsible for starting a task’s container.

High-level steps:

1. **Pull the image**
   The image is pulled from a registry like Docker Hub.

2. **Create a context**
   We use `context.Background()` to pass into Docker API calls.
   Contexts are commonly used to propagate deadlines or cancellation signals.

3. **Call ImagePull**
   The Docker client pulls the image.
   If an error occurs, it is returned immediately as a `DockerResult`.

4. **Stream output to stdout**
   The reader returned by `ImagePull` is copied to `os.Stdout` using `io.Copy`.
   This is useful for visibility into what Docker is doing.

---

## Creating the Container

Once the image is available, we prepare the configuration for `ContainerCreate`.

The Docker SDK method signature looks like this:

```go
func (cli *Client) ContainerCreate(
    ctx context.Context,
    config *container.Config,
    hostConfig *container.HostConfig,
    networkingConfig *network.NetworkingConfig,
    platform *ocispec.Platform,
    containerName string,
) (container.CreateResponse, error)
```

Important arguments:

* **container.Config**
  Holds container-level settings such as:

  * Image
  * Environment variables

* **container.HostConfig**
  Host-level settings such as:

  * Restart policy
  * Resource limits (memory)
  * Port publishing

* **network.NetworkingConfig**
  Networking details (networks, IPs, links)

* **ocispec.Platform**
  OS and CPU architecture information

* **containerName**
  Optional name for the container

---

### Mapping Our Config to Docker Types

To bridge our own `Config` struct with Docker’s types:

* Create `rp` of type `container.RestartPolicy`
* Create `r` of type `container.Resources` (mainly memory limits)
* Create `cc` of type `container.Config`
* Create `hc` of type `container.HostConfig`

In `hc`, we also set:

```text
PublishAllPorts = true
```

Instead of manually mapping ports like `-p 5432:5432`, Docker automatically exposes container ports on random available host ports. This simplifies orchestration logic significantly.

---

## Starting the Container

After creating the container, we start it using `ContainerStart`.

* Pass the container ID
* Provide empty `ContainerStartOptions` since no extra options are needed

After starting, we collect logs using `ContainerLogs` and write them to stdout using:

```text
stdcopy.StdCopy(os.Stdout, os.Stderr, out)
```

This mirrors what you’d see when running `docker run` from the CLI.

Finally, we store the container ID back into the task configuration for bookkeeping.

---

## Stop Method

The `Stop` method mirrors what `docker stop` and `docker rm` do.

Steps:

1. Call **ContainerStop** with the container ID
2. Call **ContainerRemove** with the container ID and options

Compared to `Run`, this method is intentionally simple.

---

## Take It for a Ride (main.go)

To test everything end-to-end, `main.go` wires things together.

### createContainer

At the bottom of `main.go`:

1. Define task configuration and store it in `c`
2. Create a Docker client and store it in `dc`
3. Create a `task.Docker` object `d`
4. Call `d.Run()`

---

### stopContainer

Below `createContainer`, define `stopContainer`:

* Accepts `d` as an argument
* Calls `d.Stop()`

That’s it.

---

At this stage, nothing is production-ready, but the goal isn’t perfection.
The goal is to make the orchestration flow **visible**, **understandable**, and **hackable** enough to iterate on.

This setup gives me just enough structure to move forward without getting lost in details too early.
