# Docker

**Docker** is a platform for developing, shipping, and running applications using containerization. It packages applications and their dependencies into lightweight, portable containers that run consistently across different environments.

## Table of Contents
1. [Core Concepts](#core-concepts)
2. [Container Technology Fundamentals](#container-technology-fundamentals)
3. [Images & Layers](#images--layers)
4. [Containers](#containers)
5. [Dockerfile](#dockerfile)
6. [Docker Compose](#docker-compose)
7. [Networks & Volumes](#networks--volumes)
8. [Quick Reference](#quick-reference)

---

## Core Concepts

### Container vs Virtual Machine

**Virtual Machines (VMs)**:
- Full OS virtualization (hypervisor)
- Each VM runs a complete guest OS
- Higher resource overhead (CPU, memory, disk)
- Slower startup time
- Strong isolation

**Containers**:
- OS-level virtualization (container runtime)
- Share the host OS kernel
- Lower resource overhead
- Fast startup time
- Process-level isolation

### Key Benefits

- **Consistency**: "Works on my machine" → works everywhere
- **Isolation**: Applications don't interfere with each other
- **Portability**: Run anywhere (dev, staging, production)
- **Efficiency**: Better resource utilization than VMs
- **Scalability**: Easy to scale horizontally

---

## Container Technology Fundamentals

### Linux Namespaces

Namespaces provide isolation for different aspects of the system:

- **PID namespace**: Isolated process tree (containers see only their processes)
- **Network namespace**: Isolated network stack (own network interfaces, routing tables)
- **Mount namespace**: Isolated filesystem mounts
- **UTS namespace**: Isolated hostname and domain name
- **IPC namespace**: Isolated inter-process communication
- **User namespace**: Isolated user and group IDs

### Control Groups (cgroups)

cgroups limit and account for resource usage:

- **CPU**: Limit CPU usage (shares, quotas)
- **Memory**: Limit memory usage (hard limits, soft limits)
- **I/O**: Limit disk I/O bandwidth
- **Network**: Limit network bandwidth

### Union File Systems

Layered filesystem that allows multiple filesystems to be mounted at the same time:

- **Copy-on-Write (CoW)**: Changes written to a new layer
- **Read-only layers**: Base layers remain unchanged
- **Writable layer**: Top layer for container-specific changes
- **Efficient storage**: Shared base layers across containers

### Container Runtime

Software responsible for running containers:

- **containerd**: Industry-standard container runtime (used by Docker, Kubernetes)
  - High-level runtime that manages container lifecycle
  - Handles image distribution, storage, execution
  - Implements CRI (Container Runtime Interface) for Kubernetes
  - Uses runc as low-level runtime
- **runc**: Low-level runtime that implements OCI Runtime Specification
  - Creates and runs containers according to OCI spec
  - Used by containerd, Docker, Podman
  - Direct interface to Linux namespaces and cgroups
- **CRI-O**: Kubernetes-native container runtime
  - Lightweight, OCI-compliant runtime
  - Designed specifically for Kubernetes
  - Alternative to containerd
- **Docker Engine**: Docker's runtime (uses containerd under the hood)
  - Includes Docker daemon, CLI, and containerd

### OCI (Open Container Initiative)

Standards for container formats and runtimes, enabling interoperability:

- **OCI Image Specification**: Standard image format
  - Defines image manifest, config, and layer format
  - Ensures images work across different tools (Docker, Podman, etc.)
  - Based on Docker's image format
- **OCI Runtime Specification**: Standard runtime interface
  - Defines how containers should be created and executed
  - Specifies configuration format (config.json)
  - Ensures runtimes (runc, crun, etc.) are compatible
- **Benefits**:
  - **Interoperability**: Images and containers work across tools
  - **Vendor neutrality**: Not tied to a single vendor
  - **Innovation**: Enables different implementations (runc, crun, gVisor)

---

## Images & Layers

### Image

An **image** is a read-only template used to create containers. It contains:
- Application code
- Dependencies (libraries, runtime)
- Configuration files
- OS filesystem layers

### Layers

Images are built from **layers** (read-only filesystem layers):

- Each instruction in a Dockerfile creates a new layer
- Layers are cached and reused across images
- Layers are stacked on top of each other
- Changes create new layers (copy-on-write)

**Benefits**:
- **Efficient storage**: Shared layers reduce disk usage
- **Fast builds**: Cached layers speed up rebuilds
- **Version control**: Each layer represents a change

### Image Registry

Centralized storage for Docker images:

- **Docker Hub**: Public registry (docker.io)
- **Private registries**: Harbor, AWS ECR, Google GCR, Azure ACR
- **Image tags**: Versions/labels for images (e.g., `nginx:1.21`, `app:latest`)

---

## Containers

### Container Lifecycle

1. **Created**: Container created but not started
2. **Running**: Container is executing
3. **Paused**: Container execution suspended
4. **Stopped**: Container stopped (exited)
5. **Removed**: Container deleted

### Container States

- **Running**: Container is active
- **Exited**: Container stopped (exit code 0 = success, non-zero = error)
- **Paused**: Container suspended
- **Restarting**: Container is restarting

### Container vs Image

- **Image**: Read-only template (like a class)
- **Container**: Running instance of an image (like an object)
- Multiple containers can run from the same image
- Containers have a writable layer on top of the image

---

## Dockerfile

A **Dockerfile** is a text file with instructions to build a Docker image.

### Best Practices

- **Use specific tags**: Avoid `latest` tag in production
- **Minimize layers**: Combine RUN commands where possible
- **Order instructions**: Put frequently changing instructions last (better caching)
- **Use .dockerignore**: Exclude unnecessary files from build context
- **Non-root user**: Run containers as non-root user when possible
- **Health checks**: Add HEALTHCHECK instruction
- **Multi-stage builds**: Reduce final image size

---

## Docker Compose

**Docker Compose** is a tool for defining and running multi-container Docker applications.

### Key Concepts

- **Services**: Containers defined in compose file
- **Networks**: Isolated networks for services (default network created automatically)
- **Volumes**: Persistent storage for services
- **Dependencies**: Service startup order (`depends_on`)

---

## Networks & Volumes

Network types:

- **Bridge** (default): Isolated network on single host
- **Host**: Use host's network directly (no isolation)
- **None**: No network access
- **Overlay**: Multi-host networking (Docker Swarm, Kubernetes)
- **Macvlan**: Assign MAC address to container (appears as physical device)

**Volumes** are the preferred mechanism for persistent data:

- Managed by Docker (stored in `/var/lib/docker/volumes/`)
- Survive container removal
- Can be shared between containers
- Better performance than bind mounts

---

## Quick Reference

- **Image vs Container**: Image is read-only template, container is running instance with writable layer
- **Volume vs Bind Mount**: Volume is Docker-managed, bind mount is direct host mapping
- **CMD vs ENTRYPOINT**: CMD is default command (overridable), ENTRYPOINT is fixed entry point
- **COPY vs ADD**: COPY is simpler (just copies files), ADD can also extract archives and fetch URLs
- **Bridge vs Host network**: Bridge provides isolation, host uses host network directly
- **Docker vs Podman**: Docker uses daemon, Podman is daemonless. Podman better for rootless, Docker has larger ecosystem
- **containerd vs runc**: containerd is high-level runtime (manages lifecycle), runc is low-level (creates/runs containers)
- **OCI Image Spec vs Runtime Spec**: Image Spec defines image format, Runtime Spec defines how containers execute

