# Kubernetes for Beginners: Complete Guide

A comprehensive introduction to Kubernetes concepts, from containers to production-ready deployments.

## Table of Contents

- [What is Kubernetes?](#what-is-kubernetes)
- [Core Concepts](#core-concepts)
- [Resource Hierarchy](#resource-hierarchy)
- [YAML Configuration](#yaml-configuration)
- [Networking & Services](#networking--services)
- [Microservices Architecture](#microservices-architecture)
- [Getting Started](#getting-started)
- [Real-World Example](#real-world-example)
- [Key Takeaways](#key-takeaways)

---

## What is Kubernetes?

### The Problem It Solves

**Containers** solve the problem of dependency consistency—they package your application with all dependencies into a portable unit. However, managing containers at scale requires **orchestration**.

**Kubernetes** is a container orchestration platform that:
- Automatically deploys and manages containerized applications
- Handles scaling, networking, and storage
- Ensures high availability and disaster recovery
- Manages updates without downtime

### Container Architecture Basics

Containers share the OS kernel, making them lightweight compared to VMs:
- **Containers**: 10s-100s of MB, start in milliseconds
- **VMs**: 100s of MB-GBs, start in seconds
- Both provide isolation but at different levels

### Cluster Architecture

A Kubernetes cluster consists of:
- **Master/Control Plane**: Manages the cluster (API server, scheduler, controller manager)
- **Worker Nodes**: Run containerized applications
- **kubelet**: Agent on each node ensuring containers are running
- **Container Runtime**: Docker, containerd, or CRI-O execute containers

---

## Core Concepts

### Pods

The **smallest deployable unit** in Kubernetes. A Pod can contain:
- One container (typical case)
- Multiple containers (tightly coupled, sharing network namespace)

**Key Points:**
- Containers in a Pod share the same IP address and localhost
- Short-lived and ephemeral by design
- Created and destroyed dynamically
- Always use Controllers (Deployments) to manage Pods, never directly

**Pod Lifecycle:**
```
Pending → Running → Succeeded/Failed
```

### ReplicaSets

Ensure a specified number of Pod replicas are running at any time.

**Key Points:**
- Automatically replaces failed Pods
- Enables load distribution across nodes
- Rarely created directly—Deployments manage them
- Defines desired state; controller reconciles actual state

### Deployments

Higher-level abstraction that manages ReplicaSets and provides:
- **Rolling Updates**: Gradually replace old Pods with new ones
- **Rollbacks**: Return to previous versions if issues occur
- **Version History**: Keep track of all revisions
- **Scaling**: Easily increase or decrease replicas

**Rolling Update Strategy:**
- New Pods spin up while old ones shut down
- Zero downtime updates
- Configurable speed (surge percentage, unavailable percentage)

### Services

Provide stable network endpoints for ephemeral Pods.

**Why Needed:** Pods are short-lived; their IP addresses change frequently. Services abstract this using DNS and virtual IPs.

**Service Types:**

| Type | Use Case |
|------|----------|
| **ClusterIP** | Internal pod-to-pod communication (default) |
| **NodePort** | External access via node IP + port |
| **LoadBalancer** | Cloud provider load balancer (external) |
| **ExternalName** | Map to external DNS names |

**Service Discovery:**
- Services get a stable DNS name: `service-name.namespace.svc.cluster.local`
- Automatically updated as Pods change

---

## Resource Hierarchy

```
Deployment
  └─ ReplicaSet (v1)
      └─ Pod 1
      └─ Pod 2
      └─ Pod 3

Deployment
  └─ ReplicaSet (v2) [new version]
      └─ Pod 4
      └─ Pod 5
```

**Understanding the Hierarchy:**
1. **Deployment** defines desired application state
2. **ReplicaSet** ensures correct number of Pod replicas
3. **Pods** run the actual containers

When you update a Deployment, it creates a new ReplicaSet while old one scales down.

---

## YAML Configuration

Kubernetes uses YAML for declarative configuration. Understanding YAML is essential.

### YAML Basics

**Data Types:**
```yaml
# Strings
name: "John"
quote: 'Single quoted'

# Numbers
port: 8080
percentage: 99.9

# Booleans
enabled: true
disabled: false

# Null/Empty
value: null
empty:

# Lists
ports:
  - 8080
  - 8081

# Nested objects
metadata:
  name: my-pod
  labels:
    app: web
```

### Common Kubernetes YAML Patterns

**Pod Configuration:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  labels:
    app: web
spec:
  containers:
  - name: web-container
    image: nginx:latest
    ports:
    - containerPort: 80
```

**Deployment Configuration:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: web
        image: nginx:latest
        ports:
        - containerPort: 80
```

**Service Configuration:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  type: ClusterIP
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 80
```

### YAML Best Practices

✅ Use 2-space indentation (not tabs)
✅ Quote strings with special characters
✅ Use consistent formatting
✅ Validate YAML syntax before applying
✅ Always specify resource requests/limits
✅ Use descriptive names and labels

❌ Don't use 1-space indentation
❌ Don't skip metadata
❌ Don't hardcode configuration values

---

## Networking & Services

### Kubernetes Networking Model

**Flat Network Principle:** Every Pod can communicate with every other Pod without NAT.

### Communication Patterns

**1. Pod-to-Pod (Same Node)**
```
Pod A → Pod B (via container bridge network)
```

**2. Pod-to-Pod (Different Nodes)**
```
Pod A (Node 1) → CNI Plugin → Pod B (Node 2)
```

**3. Pod-to-Service**
```
Pod A → Service (DNS: service-name.namespace.svc.cluster.local)
        → Load balanced to backing Pods
```

### Service Architecture

```
Client Request
    ↓
Service (VIP: 10.0.0.1:80)
    ↓
Endpoints (10.1.0.1:8080, 10.1.0.2:8080, 10.1.0.3:8080)
    ↓
Pod Container (running app)
```

Services maintain an Endpoints list that automatically updates as Pods change.

### Network Plugins (CNI)

Kubernetes requires a network plugin for pod-to-pod communication:

| Plugin | Characteristics |
|--------|-----------------|
| **Flannel** | Simple, lightweight, overlay network |
| **Calico** | Production-grade, network policies |
| **Weave** | Self-healing, encrypted |
| **Cilium** | High-performance, eBPF-based |

### Minikube Networking

Minikube creates a single-node cluster with built-in networking. For external access:
- Use `kubectl port-forward` for local testing
- Use `NodePort` services to expose via node IP
- Service FQDN: `service-name.default.svc.cluster.local`

### Troubleshooting Network Issues

**Check Service Endpoints:**
```bash
kubectl get endpoints service-name
```

**Test DNS Resolution:**
```bash
kubectl run -it debug --image=busybox --restart=Never -- nslookup service-name
```

**View Service Details:**
```bash
kubectl describe service service-name
```

---

## Microservices Architecture

### Monolithic vs Microservices

| Aspect | Monolithic | Microservices |
|--------|-----------|--------------|
| **Architecture** | Single codebase | Multiple independent services |
| **Deployment** | All or nothing | Deploy services independently |
| **Scalability** | Scale entire app | Scale individual services |
| **Technology** | Single stack | Heterogeneous technologies |
| **Maintenance** | Tightly coupled | Loosely coupled |

### Real-World Example: Voting Application

```
┌─────────────────────────────────────────────────────┐
│              Kubernetes Cluster                      │
├─────────────────────────────────────────────────────┤
│                                                      │
│  Voting App   Worker   Result App   Redis   PostSQL │
│  (Frontend)   (Engine) (Backend)    (Cache) (DB)   │
│                                                      │
│  Exposed via     ↑         ↑                         │
│  NodePort      Inter-service communication          │
│                 via Services                         │
└─────────────────────────────────────────────────────┘
```

**Architecture Breakdown:**

1. **Voting App** (Frontend)
   - User-facing web interface
   - Records votes via Service
   - Exposed via NodePort Service

2. **Redis** (Cache)
   - Stores in-progress votes
   - High-speed temporary storage

3. **Worker** (Backend Engine)
   - Consumes votes from Redis
   - Processes and persists to database
   - No external exposure needed

4. **Result App** (Analytics)
   - Displays voting results
   - Reads from PostgreSQL
   - Exposed via Service

5. **PostgreSQL** (Database)
   - Persistent vote storage
   - Single source of truth

### Microservices Communication

**Inter-Service Communication:**
```yaml
# Voting App → Redis
redis-service.default.svc.cluster.local:6379

# Worker → PostgreSQL
postgres-service.default.svc.cluster.local:5432

# Result App → PostgreSQL
postgres-service.default.svc.cluster.local:5432
```

### Configuration & Secrets

**ConfigMaps:** Store non-sensitive configuration
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  DATABASE_HOST: postgres-service
  DATABASE_PORT: "5432"
```

**Secrets:** Store sensitive data
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
data:
  username: base64-encoded-username
  password: base64-encoded-password
```

---

## Getting Started

### Prerequisites

- **kubectl**: Kubernetes CLI tool
- **Minikube**: Local single-node Kubernetes cluster
- **Docker**: Container runtime (optional, included with Minikube)

### Installation Steps

**1. Install Minikube**
```bash
# macOS
brew install minikube

# Linux
curl -LO https://github.com/kubernetes/minikube/releases/latest/download/minikube-linux-amd64
```

**2. Start Minikube**
```bash
minikube start
```

**3. Verify Installation**
```bash
kubectl get nodes
kubectl cluster-info
```

### Basic kubectl Commands

**Create Resources:**
```bash
kubectl apply -f deployment.yaml
kubectl run pod-name --image=nginx
```

**View Resources:**
```bash
kubectl get pods
kubectl get services
kubectl get deployments
kubectl describe pod pod-name
```

**Debugging:**
```bash
kubectl logs pod-name
kubectl exec -it pod-name -- /bin/bash
kubectl events
```

**Port Forwarding:**
```bash
kubectl port-forward service/web-service 8080:80
```

**Delete Resources:**
```bash
kubectl delete pod pod-name
kubectl delete -f deployment.yaml
```

---

## Real-World Example

### Deploy Voting Application

**Step 1: Clone/Prepare Files**
```bash
cd kubernetes-intro/voting-app
```

**Step 2: Deploy Services & Deployments**
```bash
# Deploy all components
kubectl apply -f .

# Verify deployments
kubectl get all
```

**Step 3: Access Applications**
```bash
# Voting App (port 80 → frontend)
kubectl port-forward svc/voting-service 8080:80

# Result App (port 5000 → results)
kubectl port-forward svc/result-service 3000:5000

# Open browser to localhost:8080 and localhost:3000
```

**Step 4: Verify Inter-Service Communication**
```bash
# Check if services can reach each other
kubectl exec -it <pod-name> -- curl http://redis-service:6379

# Check logs
kubectl logs <deployment-name>
```

**Step 5: Test Scaling**
```bash
# Scale voting app
kubectl scale deployment voting-app --replicas=5

# Watch pods being created
kubectl get pods --watch
```

**Step 6: Cleanup**
```bash
kubectl delete -f .
minikube stop
```

---

## Key Takeaways

### Core Principles

1. **Declarative Configuration**: Define desired state in YAML; Kubernetes reconciles it
2. **Immutable Infrastructure**: Update by deploying new versions, not modifying existing
3. **Self-Healing**: Controllers automatically recover from failures
4. **Scalability**: Easily scale workloads horizontally
5. **Loose Coupling**: Services abstract pod implementation details

### Essential Patterns

| Pattern | Purpose |
|---------|---------|
| **Deployment** | Deploy and manage versioned applications |
| **Service** | Stable network endpoint for Pods |
| **ConfigMap** | Inject configuration without rebuilding images |
| **Secret** | Securely store sensitive data |
| **Namespace** | Logical cluster partitioning |
| **Labels** | Organize and select resources |

### Workflow for New Applications

```
1. Containerize application (Docker image)
2. Create Deployment manifest (YAML)
3. Create Service manifest for networking
4. Add ConfigMap/Secrets for configuration
5. Apply manifests (kubectl apply -f .)
6. Verify and test (kubectl logs, describe, exec)
7. Scale and monitor as needed
```

### Common Mistakes to Avoid

❌ Creating Pods directly (use Deployments)
❌ Hardcoding configuration in images
❌ Exposing all services externally (use appropriate Service types)
❌ Not setting resource requests/limits
❌ Ignoring security best practices

### What's Next?

- **Persistent Storage**: StatefulSets, PersistentVolumes, PersistentVolumeClaims
- **Advanced Networking**: Ingress, Network Policies
- **Security**: RBAC, Pod Security Policies, Network Segmentation
- **Observability**: Logging, Monitoring, Tracing
- **Production**: Multi-node clusters, High Availability, Disaster Recovery
- **Package Management**: Helm for templating and chart management

---

## Resources

- Official Documentation: https://kubernetes.io/docs/
- Interactive Tutorial: https://kubernetes.io/docs/tutorials/
- Kubectl Cheat Sheet: https://kubernetes.io/docs/reference/kubectl/cheatsheet/
- Playground: https://www.katacoda.com/courses/kubernetes

---

## Quick Reference Diagram

```
┌─────────────────────────────────────────────────────┐
│           Your Application Code (Docker Image)      │
└──────────────────────┬──────────────────────────────┘
                       ↓
┌─────────────────────────────────────────────────────┐
│              Deployment (desired state)             │
│  - Replicas: 3                                      │
│  - Image: your-app:v1.0                             │
│  - Configuration via ConfigMap/Secrets              │
└──────────────────────┬──────────────────────────────┘
                       ↓
┌──────────────────────────────────────────────────────┐
│       ReplicaSet (maintains 3 Pod replicas)         │
│  ┌──────────┬──────────┬──────────┐                 │
│  │   Pod 1  │   Pod 2  │   Pod 3  │                 │
│  └────────┬─┴────────┬─┴────────┬─┘                 │
└───────────┼──────────┼──────────┼────────────────────┘
            ↓          ↓          ↓
      ┌──────────────────────────────────┐
      │    Service (stable endpoint)     │
      │  DNS: service-name.default...    │
      │  Load balances traffic to Pods   │
      └──────────────────────────────────┘
            ↓
      External Access (NodePort/LB)
```

---

**Happy Kubernetes Learning!** 🚀

For detailed information on specific topics, refer to the individual markdown files in this directory.
