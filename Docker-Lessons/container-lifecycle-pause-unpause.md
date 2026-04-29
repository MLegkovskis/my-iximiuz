# Docker Container Lifecycle: Pause and Unpause

## Challenge Goal

The goal of this challenge was to start a Docker container, pause it so the process inside was suspended, inspect the container to identify its exact status, and then unpause it so the application resumed.

The target image was:

```bash
ghcr.io/iximiuz/labs/resource-hog/herder:v1.0.0
```

## Main Idea

A container is a managed environment around Linux processes. Docker gives you lifecycle controls around that environment: start it, inspect it, pause it, unpause it, stop it, and remove it.

Pausing a container freezes the processes inside without destroying the container. That is useful when you want to temporarily suspend work but keep the application state available to resume.

## What is a Container?

A **container** is a packaged application environment. It contains the app, its dependencies, and enough filesystem structure to run consistently on a machine with a container runtime such as Docker.

At a high level, a container is not a full virtual machine. It uses the host Linux kernel, but Docker gives it isolation around things like:

- Processes
- Filesystems
- Networking
- Resource usage

Containers are useful because they make applications easier to run, repeat, move, and clean up.

## What is Docker?

**Docker** is a tool for working with containers. It can download container images, start containers, list running containers, inspect their state, pause them, unpause them, stop them, and remove them.

In this lesson, Docker was the command-line tool we used to control the `herder` container.

## What is an Image?

A **container image** is the template used to create a container. It contains the application and the filesystem needed to run it.

In this challenge, the image was:

```bash
ghcr.io/iximiuz/labs/resource-hog/herder:v1.0.0
```

When we ran `docker run`, Docker pulled this image first because it was not already present on the server.

## Container vs Process

A **process** is a running program on Linux. For example, `bash`, `nginx`, or a Python script can all be processes.

A **container** usually runs one main process, but Docker wraps that process with extra isolation and management. The process still runs on the host kernel, but it sees a controlled environment instead of the full host system.

Simple way to think about it:

- A process is the running program.
- A container is a managed, isolated environment around one or more processes.
- Docker is the tool used to create, start, stop, inspect, pause, and remove containers.

## New Principle: Pausing a Container

Pausing a container suspends its processes without stopping or killing them.

When a container is paused:

- The container still exists.
- Its processes are frozen.
- The app does not continue doing work.
- It can be resumed later with `docker unpause`.

This is different from stopping a container. Stopping asks the process to exit; pausing keeps it in memory but prevents it from running.

## Commands Run

These were the commands run on the remote iximiuz playground server.

### 1. Connect to the Server

```bash
ssh -i /home/mark/.ssh/iximiuz_labs_user -p 48568 laborant@127.0.0.1
```

Why:

- `ssh` connects to the remote Linux playground.
- `-i /home/mark/.ssh/iximiuz_labs_user` uses the private key for authentication.
- `-p 48568` connects to the forwarded SSH port.
- `laborant@127.0.0.1` is the remote user and host.

In practice, commands were run through SSH from the local machine.

### 2. Start the Container

```bash
docker run -d --name herder ghcr.io/iximiuz/labs/resource-hog/herder:v1.0.0
```

What this means:

- `docker run` creates and starts a new container.
- `-d` runs it in the background.
- `--name herder` gives the container an easy name.
- The final value is the image to run.

Why:

This created a new container named `herder` from the challenge image and started it.

The image was pulled automatically because it was not already available locally.

### 3. Check the Container is Running

```bash
docker ps --filter name=herder
```

This lists running containers matching the name `herder`.

The container showed as `Up`, meaning it was running.

Why:

Before pausing the container, we needed to confirm that it had started successfully.

### 4. Pause the Container

```bash
docker pause herder
```

This suspended the processes inside the `herder` container.

Why:

The challenge required the container to stay alive but have its processes suspended. `docker pause` does that. It does not delete, stop, or restart the container.

### 5. Inspect the Container Status

```bash
docker inspect --format "{{.State.Status}}" herder
```

Output:

```text
paused
```

This was the exact status required by the challenge.

Why:

`docker inspect` shows detailed metadata about a container. The `--format` option extracts only the status field instead of printing the full JSON output.

The important point is that the container had to remain paused while this command was run, because the challenge wanted the exact status while suspended.

### 6. Unpause the Container

```bash
docker unpause herder
```

This resumed the suspended processes.

Why:

The final challenge step was to resume the application. `docker unpause` allows the frozen processes to continue running.

### 7. Confirm it is Running Again

```bash
docker inspect --format "{{.State.Status}}" herder
```

Output:

```text
running
```

Why:

This confirmed the container had moved from `paused` back to `running`.

## Exact Command Sequence Used

The full challenge flow can be repeated with:

```bash
docker run -d --name herder ghcr.io/iximiuz/labs/resource-hog/herder:v1.0.0
docker ps --filter name=herder
docker pause herder
docker inspect --format "{{.State.Status}}" herder
docker unpause herder
docker inspect --format "{{.State.Status}}" herder
```

Expected important outputs:

```text
paused
running
```

## Useful Docker Commands from this Lesson

```bash
docker run -d --name <container-name> <image>
docker ps
docker ps --filter name=<container-name>
docker pause <container-name>
docker inspect <container-name>
docker inspect --format "{{.State.Status}}" <container-name>
docker unpause <container-name>
```

## New Principles Learned

- A container is a managed environment around one or more Linux processes.
- A container image is the template used to create a container.
- `docker run` starts a container from an image.
- `docker ps` lists running containers.
- `docker pause` freezes the processes inside a running container.
- `docker inspect` shows the container's state and metadata.
- `docker inspect --format "{{.State.Status}}"` prints only the status value.
- `docker unpause` resumes a paused container.
- Pausing is different from stopping: pausing suspends work, stopping exits the workload.

## Conclusion

In this lesson, we started a Docker container, paused it, inspected its state, and unpaused it again.

The key takeaway is that Docker controls normal Linux processes by placing them inside a managed container environment. `docker pause` freezes the workload without destroying it, `docker inspect` lets us confirm the exact state, and `docker unpause` resumes the workload.

For this challenge, the exact status answer was:

```text
paused
```
