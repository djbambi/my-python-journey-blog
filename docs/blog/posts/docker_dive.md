---
title: Docker Deep(ish) Dive
date: 2026-01-20
categories:
  - Docker
authors:
  - david
tags:
  - pybites
  - pdm
description: >
  I didn't just want to get images and run containers, I wanted to understand things a bit more for when it all goes wrong.
---

# Docker Deep(ish) Dive

I recently did a Docker course on Udemy, but the real learning didn’t come from watching videos — it came from **building, breaking, and rebuilding** things myself.

What I thought Docker was and what it *actually is* turned out to be subtly but importantly different.

---

## Containers are not mini computers

The biggest mental shift for me was realising that a Docker container is **not a lightweight VM**.

A container:
- does **not** have its own kernel
- does **not** boot an operating system
- **is** just one or more Linux processes, isolated by the kernel

On Linux, those processes run directly on the host kernel.  
On macOS and Windows, Docker Desktop quietly runs a **single Linux VM**, and all containers run as Linux processes *inside that VM*. The VM isn’t the point — it’s just there because macOS and Windows don’t have a Linux kernel.

That distinction matters.

---

## Images provide user-space, not an OS

When you write:

```Dockerfile
FROM python:3.12-slim
```

you’re not getting an operating system — you’re getting:
- a Linux **user-space filesystem**
- a Python binary
- shared libraries Python depends on

The kernel always comes from the host (or the Docker Desktop VM). Containers *share* it.

That’s why containers start instantly and why you can run hundreds of them without “booting” anything.

---

## Containers have a filesystem — but it’s disposable

A container has a filesystem view built from:
- read-only image layers
- a thin writable layer on top

That writable layer disappears when the container is removed.

This explains a lot:
- why logs “vanish” when a container exits
- why batch jobs don’t hang around
- why inspecting container filesystems is usually the wrong approach

If you care about data, you **mount volumes**.

Once I started mounting `data/` and `logs/` from the host, Docker stopped feeling mysterious and started feeling *mechanical* — in a good way.

---

## `docker exec` is not SSH

Another subtle but important insight:  
`docker exec` doesn’t connect over the network. It just asks the kernel to start **another process** inside the same container namespaces.

That’s why it’s instant, and why it only works when the container is running.

---

## Dockerfiles vs docker-compose

The distinction finally clicked:

- **Dockerfile**: *What is this thing?*  
  (base image, dependencies, code, default command)

- **docker-compose**: *How does it run?*  
  (env vars, volumes, ports, multiple containers)

Once I separated “build” from “run”, things stopped getting tangled.

---

## Why this is such a cool technology

Docker feels powerful not because it’s magical, but because it’s **honest**:
- processes, not machines
- explicit filesystem access
- disposable by default
- reproducible by design

It forces good habits — especially around stateless execution and externalised data — and once you internalise the model, a lot of modern infrastructure suddenly makes sense.

I went in expecting “containers”.  
I came out understanding **processes, kernels, filesystems, and isolation** a bit better.

That’s a pretty good trade.
