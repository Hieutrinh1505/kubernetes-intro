# Kubernetes Overview

## Problem Containers Solve

Containers solve the **dependency and environment consistency** problem across different machines. They allow applications and their dependencies to be packaged together, eliminating the "works on my machine" problem.

## Understanding Operating Systems and Containers

### Operating System Structure
- An OS consists of a **kernel** (core) and **software/utilities** built on top of it
- The kernel manages hardware resources and enables communication between applications and hardware

### Docker Containers
- Docker containers **share the underlying OS kernel** rather than running their own
- **Example**: On Ubuntu, Docker containers use the Ubuntu/Linux kernel, not a separate one
- This is why containers are lightweight—they don't duplicate the entire OS

### Special Case: Docker on Windows
- Windows doesn't have a native Linux kernel, so Docker runs a lightweight **Linux VM under the hood**
- This VM provides the Linux kernel that containers need

## Containers vs. Virtual Machines

| Aspect | Containers | Virtual Machines |
|--------|-----------|------------------|
| **Kernel** | Share host OS kernel | Have their own kernel |
| **Size** | Lightweight (MBs) | Heavy (GBs) |
| **Disk Space** | Minimal overhead | Significant overhead |
| **Boot Time** | Seconds | Minutes |
| **Resource Usage** | Efficient | High overhead |
| **Hypervisor** | Not needed | Required |

**Key Difference**: VMs include an entire OS with their own software stack, while containers only package the application and its dependencies.

### Docker Images
- A Docker **image** is a **template/blueprint** for creating containers
- Think of it as a snapshot that includes your application and all dependencies
- Multiple containers can be created from the same image

## Key Takeaways

1. **Containers solve consistency**: Same environment across dev, test, and production
2. **Lightweight and fast**: Containers share the OS kernel, making them efficient
3. **Docker images are reusable templates**: One image can spawn many container instances
4. **Containers > VMs for most use cases**: Much faster startup and resource usage
5. **Foundation for Kubernetes**: Kubernetes orchestrates these lightweight containers at scale
## Container Orchestration

### What is Container Orchestration?

Container orchestration automates the deployment, management, and scaling of containers across multiple machines. Key orchestration tools include:
- **Kubernetes** (most popular)
- **Docker Swarm**
- **Apache Mesos**

**Why needed**: Managing hundreds/thousands of containers manually is impossible. Orchestration tools handle:
- Scaling containers up/down based on demand
- Distributing workloads across multiple machines
- Self-healing (replacing failed containers)
- Rolling updates and rollbacks

### Kubernetes Architecture

#### Cluster Components

A Kubernetes cluster consists of:
- **Master Node(s)** - Control plane that manages the cluster
- **Worker Nodes** - Machines that run the actual containers

#### Master Node Components

| Component | Responsibility |
|-----------|-----------------|
| **API Server** | Exposes Kubernetes API; all cluster communication goes through it |
| **etcd** | Key-value store that stores all cluster state and configuration; ensures data consistency |
| **Scheduler** | Watches for new Pods and assigns them to appropriate worker nodes |
| **Controller Manager** | Runs controller processes (e.g., monitoring if desired containers are running and restarting failed ones) |

#### Worker Node Components

| Component | Responsibility |
|-----------|-----------------|
| **Kubelet** | Agent that ensures containers are running in pods as expected |
| **Container Runtime** | Software that runs containers (Docker, containerd, etc.) |
| **kube-proxy** | Network proxy that maintains network rules and load balancing |

### Key Concepts

- **Pod**: Smallest deployable unit in Kubernetes; usually contains one container (can contain multiple)
- **Node**: Physical or virtual machine that runs pods
- **Cluster**: Set of nodes managed by a master node

### kubectl - Command Line Tool

**kubectl** is the command-line interface to interact with Kubernetes clusters.

Common commands:
```bash
kubectl run <pod-name>           # Deploy a pod
kubectl get nodes               # List all nodes in the cluster
kubectl cluster-info            # Display cluster information
kubectl get pods                # List all pods
kubectl describe pod <name>     # Get details about a specific pod
```

**Example**: `kubectl run hello-minikube --image=hello-minikube`

## Key Takeaways

1. **Orchestration is essential**: Managing containers at scale requires automation
2. **Master-Worker architecture**: Master controls, workers execute
3. **Key components work together**: API Server, etcd, Scheduler, and Controllers manage the cluster
4. **kubectl is your interface**: All cluster operations go through kubectl commands
5. **Pods are the unit of deployment**: Not containers directly—Kubernetes works with pods


---

## Container Runtimes: Docker vs containerd

### History and Evolution

Container runtimes have evolved significantly in the Kubernetes ecosystem:

1. **Early Days**: Kubernetes originally used Docker as the only container runtime
2. **CRI (Container Runtime Interface)**: Kubernetes introduced CRI to support multiple runtimes
3. **Dockershim**: A compatibility layer that allowed Docker to work with Kubernetes despite not natively implementing CRI
4. **Dockershim Removal**: In Kubernetes v1.24 (2022), dockershim was removed, ending direct Docker support

### Docker vs containerd

| Aspect | Docker | containerd |
|--------|--------|------------|
| **Scope** | Full development platform | Lightweight runtime only |
| **Components** | CLI, API, Build, Volumes, Auth, Security, Runtime | Runtime only |
| **Size** | Large (includes many tools) | Minimal (just what's needed to run containers) |
| **Kubernetes** | No longer directly supported | Native CRI support |
| **Best for** | Local development | Production Kubernetes clusters |

**Key Point**: containerd is actually a component that was **extracted from Docker**. When you use Docker, you're using containerd underneath.

### Why Kubernetes Dropped Docker

- Docker has too many features Kubernetes doesn't need (build, volumes, auth, etc.)
- containerd is lighter and more efficient
- Docker doesn't implement CRI natively (required dockershim)
- **Your containers still work**: Docker images run perfectly on containerd

### Container Runtime CLI Tools

Different runtimes come with different command-line tools:

#### ctr (containerd CLI)
```bash
ctr images pull docker.io/library/nginx:latest
ctr run docker.io/library/nginx:latest my-nginx
```
**Note**: Low-level tool, not very user-friendly

#### nerdctl (Docker-compatible CLI for containerd)
```bash
nerdctl run -d -p 80:80 nginx        # Same syntax as Docker!
nerdctl ps
nerdctl build -t myapp:latest .
```
**Best choice**: User-friendly, supports Docker Compose, same commands as Docker

#### crictl (CRI debugging tool)
```bash
crictl ps                   # List containers
crictl pods                 # List pods
crictl images               # List images
```
**Purpose**: Designed for **debugging Kubernetes**, not general container management

### Command Comparison

| Operation | Docker | containerd (nerdctl) | CRI (crictl) |
|-----------|--------|---------------------|--------------|
| **List containers** | `docker ps` | `nerdctl ps` | `crictl ps` |
| **Run container** | `docker run` | `nerdctl run` | N/A (use kubectl) |
| **List images** | `docker images` | `nerdctl images` | `crictl images` |
| **Pull image** | `docker pull` | `nerdctl pull` | `crictl pull` |

## Key Takeaways

1. **containerd is the future**: Kubernetes uses containerd (or other CRI runtimes), not Docker
2. **Your images still work**: Docker-built images run perfectly on containerd-based clusters
3. **Use nerdctl for development**: Drop-in replacement for Docker CLI with containerd
4. **crictl is for debugging**: Use it to inspect Kubernetes container runtime issues
5. **Docker isn't gone**: Still excellent for local development and building images