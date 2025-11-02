# Kubernetes Services

## Overview

A **Service** is a Kubernetes object that provides a stable endpoint for accessing Pods. Services abstract away the ephemeral nature of Pods and provide consistent networking for applications.

**Key Problem Services Solve**:
- Pods are ephemeral (can be created/destroyed at any time)
- Pod IPs change when Pods are replaced
- Applications need stable, predictable endpoints
- Services solve this by providing a stable IP and DNS name

---

## Why Do We Need Services?

### Without Services (❌ Problematic)

```
Pod A (10.1.1.5) crashes → Deployment creates new Pod
New Pod B (10.2.2.9) with different IP

Other applications still point to old IP (10.1.1.5) → Connection fails ❌
```

### With Services (✅ Solution)

```
Pod A (10.1.1.5) crashes → Deployment creates new Pod B (10.2.2.9)
Service still has stable IP (10.96.0.1)
Other applications connect to Service IP → Always works ✅
```

---

## Service Architecture

### Basic Service Components

```
┌─────────────────────────────────────┐
│         Service                     │
│  IP: 10.96.0.1                      │
│  Port: 80                           │
│  Name: web-service                  │
│  Selector: app=web                  │
└────────────┬────────────────────────┘
             │ Load Balancing
    ┌────────┼────────┐
    ▼        ▼        ▼
  Pod A    Pod B    Pod C
 10.1.1.5 10.1.1.6 10.2.2.8
:8080      :8080     :8080
```

### Service Key Concepts

1. **Selector**: Labels that identify which Pods this Service manages
2. **Port**: Port number on the Service (what external apps use)
3. **TargetPort**: Port on the Pod container (where app listens)
4. **NodePort**: (optional) External port on the node (30000-32767)

---

## Service YAML Structure

### Basic Service Definition

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
  namespace: default
  labels:
    app: web
spec:
  type: ClusterIP                    # Service type (default)
  selector:
    app: web                         # Match Pods with this label
  ports:
    - name: http                     # Port name (optional)
      protocol: TCP                  # Protocol (TCP/UDP)
      port: 80                       # Service port
      targetPort: 8080               # Pod container port
```

---

## Service Types

### 1. ClusterIP (Default)

**Purpose**: Internal communication within the cluster

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
    - port: 80
      targetPort: 8080
```

**Characteristics**:
- Only accessible from within cluster
- Stable internal IP address
- Pods access via service name or FQDN
- Suitable for backend services
- **Most common type** for microservice architectures

**Access Methods**:
```bash
# From another Pod in same namespace
curl http://backend-service:80

# From another Pod in different namespace
curl http://backend-service.production.svc.cluster.local:80

# Using service IP directly
curl http://10.96.0.1:80
```

**Use Cases**:
- Backend services (databases, APIs)
- Internal microservices
- Services not exposed to external traffic
- Grouping Pods for microservice communication

### ClusterIP for Microservices

ClusterIP is the **foundation of microservice architecture** in Kubernetes:

**Microservice Pattern**:
```
Frontend Pods
    │
    ├─ calls via ClusterIP → Backend Service
    │                           │
    │                           ├─ Backend Pod 1
    │                           ├─ Backend Pod 2
    │                           └─ Backend Pod 3
    │
    └─ calls via ClusterIP → Database Service (PostgreSQL)
                                │
                                └─ Database Pod
```

**Benefits for Microservices**:
1. **Service isolation**: Each microservice has its own Service
2. **Load distribution**: Service distributes requests across Pod replicas
3. **Pod abstraction**: Consumers don't know about individual Pods
4. **Service discovery**: DNS automatically resolves service names
5. **Loose coupling**: Services communicate via stable endpoints, not Pod IPs

**Example**: E-commerce microservices
```yaml
# Frontend Service (accessed externally via LoadBalancer)
---
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
spec:
  type: LoadBalancer
  selector:
    app: frontend
  ports:
    - port: 80
      targetPort: 3000

# Backend API Service (accessed internally via ClusterIP)
---
apiVersion: v1
kind: Service
metadata:
  name: api-service
spec:
  type: ClusterIP
  selector:
    app: api
  ports:
    - port: 80
      targetPort: 5000

# Database Service (accessed internally via ClusterIP)
---
apiVersion: v1
kind: Service
metadata:
  name: postgres-service
spec:
  type: ClusterIP
  selector:
    app: postgres-db
  ports:
    - port: 5432
      targetPort: 5432
```

Frontend Pods can access API via: `http://api-service:80`
API Pods can access Database via: `http://postgres-service:5432`

---

### 2. NodePort

**Purpose**: Expose service to external traffic via node IP and port

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  type: NodePort
  selector:
    app: web
  ports:
    - port: 80                   # Service port (cluster internal)
      targetPort: 8080           # Pod container port
      nodePort: 30001            # Node port (30000-32767)
```

**How NodePort Works**:

```
External Client (192.168.1.100)
    │
    ├─ curl http://192.168.49.2:30001
    │
    ▼
Node IP:30001 (192.168.49.2:30001)
    │
    ▼
Service IP:80 (10.96.0.1:80)
    │
    ├─ Load Balanced to ─┐
    │                    │
    ▼                    ▼
Pod A:8080            Pod B:8080
(10.1.1.5:8080)      (10.1.1.6:8080)
```

**Port Mapping Explained**:

| Component | Port | Purpose |
|-----------|------|---------|
| **NodePort** | 30001 | External access point (node level) |
| **Port** | 80 | Service port (cluster level) |
| **TargetPort** | 8080 | Container application port |

**Flow**:
```
30001 (external) → 80 (service) → 8080 (pod container)
```

**Characteristics**:
- Accessible from outside cluster
- Port range: 30000-32767
- Kubernetes auto-assigns port if not specified
- Works on all nodes

**Access Methods**:
```bash
# From external client
curl http://<node-ip>:30001
curl http://192.168.49.2:30001

# From kubectl (tunneling)
kubectl port-forward service/web-service 8080:80
# Then access: localhost:8080
```

**Use Cases**:
- Development and testing
- Exposing to external networks
- Testing from local machine
- Demo and prototype applications

---

### 3. LoadBalancer

**Purpose**: Provision external load balancer (cloud providers)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  type: LoadBalancer
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 8080
```

**How LoadBalancer Works**:

```
Internet (external clients)
    │
    ├─ 203.0.113.45:80 (Cloud Load Balancer)
    │
    ▼
Cloud LB routes to NodePort service
    │
    ├─┐ Nodes (multiple)
    │ │
    ▼ ▼
  Service → Pods
```

**Characteristics**:
- Cloud provider auto-provisions load balancer
- Automatically creates NodePort internally
- External IP assigned by cloud provider
- True load balancing across multiple nodes

**Supported Cloud Providers**:
- AWS (ELB/NLB)
- GCP (Cloud Load Balancer)
- Azure (Azure Load Balancer)
- DigitalOcean, Linode, etc.

**Access Methods**:
```bash
# Get the external IP
kubectl get service web-service
# Shows: EXTERNAL-IP: 203.0.113.45

# Access from anywhere
curl http://203.0.113.45:80
```

**Use Cases**:
- Production deployments
- Public APIs
- Web applications
- Services requiring real load balancing

**On Minikube** (no cloud provider):
```bash
kubectl get service
# Shows: EXTERNAL-IP: <pending>
# Use minikube tunnel or port-forward instead
minikube tunnel
```

---

### 4. ExternalName

**Purpose**: Map Kubernetes service to external DNS name

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-db
spec:
  type: ExternalName
  externalName: database.example.com
  ports:
    - port: 5432
```

**How ExternalName Works**:

```
Pod needs to connect to external database
    │
    ▼
kubectl exec pod -- psql -h external-db.default.svc.cluster.local
    │
    ▼
Kubernetes DNS resolves to: database.example.com
    │
    ▼
External database server (outside cluster)
```

**Characteristics**:
- No load balancing (simple CNAME record)
- DNS redirection only
- Internal IP not assigned
- Useful for external service abstraction

**Use Cases**:
- External databases
- Third-party APIs
- Legacy systems outside cluster
- Abstracting external dependencies

---

## Service Type Comparison

| Feature | ClusterIP | NodePort | LoadBalancer | ExternalName |
|---------|-----------|----------|--------------|--------------|
| **Internal Access** | ✅ Yes | ✅ Yes | ✅ Yes | ✅ Yes |
| **External Access** | ❌ No | ✅ Yes (Node) | ✅ Yes (LB) | ✅ Via DNS |
| **Load Balancing** | ✅ Yes | ✅ Yes | ✅ Yes (Cloud) | ❌ No |
| **Stable IP** | ✅ Yes | ✅ Yes | ✅ Yes | ❌ No |
| **Cloud LB Required** | ❌ No | ❌ No | ✅ Yes | ❌ No |
| **Default** | ✅ Yes | ❌ No | ❌ No | ❌ No |

---

## Service Discovery

### Method 1: Service Name (DNS)

```bash
# From another Pod in same namespace
curl http://web-service:80

# From another Pod in different namespace
curl http://web-service.production.svc.cluster.local:80

# Environment variable (legacy)
$WEB_SERVICE_HOST
$WEB_SERVICE_PORT
```

### Method 2: Service IP (Direct)

```bash
# Get service IP
kubectl get service web-service
# Copy the CLUSTER-IP: 10.96.0.1

# Use directly
curl http://10.96.0.1:80
```

### Method 3: FQDN (Fully Qualified Domain Name)

```
Format: <service-name>.<namespace>.svc.cluster.local

Example: web-service.default.svc.cluster.local
         backend-api.production.svc.cluster.local
```

---

## Service Selector and Load Balancing

### How Selectors Work

Services use **label selectors** to identify Pods:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  selector:
    app: web              # Match Pods with label: app=web
    env: production       # AND label: env=production
  ports:
    - port: 80
      targetPort: 8080
```

**Pod Matching**:

```yaml
# This Pod matches (has both labels)
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: web
    env: production
    tier: frontend

# This Pod does NOT match (missing env label)
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: web
    tier: frontend
```

### Endpoints

Services automatically create **Endpoints** that list all matching Pods:

```bash
kubectl get endpoints web-service
# Shows all Pods the service routes to

kubectl describe service web-service
# Shows Endpoints section with Pod IPs
```

---

## Creating Services

### Method 1: Imperative (Quick)

```bash
# ClusterIP (default)
kubectl expose pod web-pod --port=80 --target-port=8080

# NodePort
kubectl expose pod web-pod --type=NodePort --port=80 --target-port=8080

# LoadBalancer
kubectl expose deployment web --type=LoadBalancer --port=80 --target-port=8080

# Specify NodePort
kubectl expose pod web-pod --type=NodePort --port=80 --target-port=8080 --node-port=30001
```

### Method 2: Declarative (Best Practice)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
  labels:
    app: web
spec:
  type: ClusterIP
  selector:
    app: web
  ports:
    - name: http
      port: 80
      targetPort: 8080
      protocol: TCP
```

Apply:
```bash
kubectl create -f service.yaml
```

---

## Common Service Scenarios

### Scenario 1: Exposing Frontend to Internet

**Requirement**: Frontend app needs to be accessible from internet

**Solution**: Use LoadBalancer or NodePort

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
spec:
  type: LoadBalancer          # Or NodePort for development
  selector:
    app: frontend
  ports:
    - port: 80
      targetPort: 3000
```

### Scenario 2: Backend-to-Backend Communication

**Requirement**: Frontend Pods need to access Backend Pods

**Solution**: Use ClusterIP (default)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
    - port: 80
      targetPort: 5000
```

**From Frontend Pod**:
```bash
curl http://backend-service:80/api/users
```

### Scenario 3: Database Access

**Requirement**: All Pods need to access database

**Solution**: Use ExternalName (if external) or ClusterIP (if Pod-based)

```yaml
# For external database
apiVersion: v1
kind: Service
metadata:
  name: postgres-db
spec:
  type: ExternalName
  externalName: postgres.aws.amazon.com
  ports:
    - port: 5432

# From Pod
# psql -h postgres-db.default.svc.cluster.local -U user
```

### Scenario 4: Development Testing

**Requirement**: Test service locally from development machine

**Solution**: Use NodePort or port-forward

```bash
# Option 1: NodePort
kubectl expose deployment web --type=NodePort --port=80
kubectl get service web
# Access: http://<minikube-ip>:30001

# Option 2: Port-forward (simpler)
kubectl port-forward service/web 8080:80
# Access: http://localhost:8080
```

---

## Service Management Commands

```bash
# List services
kubectl get services
kubectl get svc                           # Short form

# Detailed service info
kubectl describe service <name>
kubectl describe svc <name>

# Service endpoints (which Pods are selected)
kubectl get endpoints <service-name>

# Create service
kubectl create -f service.yaml
kubectl expose deployment <name> --type=NodePort --port=80

# Update service
kubectl edit service <name>
kubectl patch service <name> -p '{"spec":{"type":"LoadBalancer"}}'

# Delete service
kubectl delete service <name>

# Port forwarding
kubectl port-forward service/<name> 8080:80

# Check service IP and ports
kubectl get service <name> -o yaml
```

---

## Service Best Practices

1. **Always use Services for Pod access**
   - ❌ Don't: `curl 10.1.1.5:8080` (Pod IP)
   - ✅ Do: `curl web-service:80` (Service)

2. **Use descriptive service names**
   - ✅ Good: `backend-api`, `postgres-db`, `cache-redis`
   - ❌ Poor: `service1`, `service2`

3. **Match labels carefully**
   - Make sure Pod labels match Service selector
   - Test with: `kubectl get pods --show-labels`

4. **Use appropriate service types**
   - ClusterIP for internal only
   - NodePort for dev/testing
   - LoadBalancer for production
   - ExternalName for external services

5. **Document port mappings**
   - Use meaningful port names
   - Add comments explaining port purposes

6. **Separate frontend and backend services**
   - Frontend: LoadBalancer/NodePort
   - Backend: ClusterIP only
   - Database: ClusterIP or ExternalName

7. **Monitor endpoints**
   - Regularly check: `kubectl get endpoints <service>`
   - Ensure Pods are in endpoints
   - If empty, check Pod labels and selectors

---

## Troubleshooting Services

### Service Has No Endpoints

**Problem**: `kubectl get endpoints <service>` shows empty

**Causes and Solutions**:

```bash
# 1. Check Pod labels
kubectl get pods --show-labels
# Verify labels match service selector

# 2. Check service selector
kubectl describe service <name>
# Look at "Selector" section

# 3. Create matching labels if needed
kubectl label pod <pod-name> app=web
```

### Can't Access Service from External

**Problem**: External client can't reach NodePort/LoadBalancer

**Causes and Solutions**:

```bash
# 1. Verify service type
kubectl get service <name>
# Ensure TYPE is NodePort or LoadBalancer

# 2. Check NodePort range
# Must be 30000-32767
kubectl get service <name> -o yaml | grep nodePort

# 3. Verify firewall allows port
# On host/cloud: ensure port is open

# 4. Check node IP
kubectl get nodes -o wide
# Use INTERNAL-IP for NodePort access
```

### Service Works but Slow

**Problem**: Service responding slowly

**Causes and Solutions**:

```bash
# 1. Check Pod status
kubectl get pods
# Ensure Pods are Running

# 2. Check Pod resources
kubectl describe pod <name>
# Look for resource constraints

# 3. Check load distribution
kubectl get endpoints <service> -o yaml
# Verify all Pods are in endpoints

# 4. Check node connectivity
kubectl get nodes
# Ensure all nodes are Ready
```

---

## Key Takeaways

1. **Services solve the Pod IP problem**: Provide stable endpoint for accessing Pods
2. **Four service types**: ClusterIP (internal), NodePort (external node), LoadBalancer (cloud LB), ExternalName (external DNS)
3. **Port mapping**: NodePort (external) → Port (service) → TargetPort (pod)
4. **Selectors are critical**: Services use labels to identify Pods
5. **ClusterIP is default**: Most common for internal communication
6. **NodePort range**: 30000-32767 (industry standard)
7. **Load balancing**: Services distribute traffic across all matching Pods
8. **Service discovery**: Access via name (DNS) or FQDN
9. **Endpoints list Pods**: Verify with `kubectl get endpoints`
10. **Development trick**: Use `kubectl port-forward` for local testing

### ClusterIP for Microservices - Summary

ClusterIP Services are the **foundation of microservice communication** in Kubernetes:
- **Group Pods together**: Services group related Pods for easy access
- **Enable microservices**: Each microservice has its own ClusterIP Service
- **Load balanced**: Requests distributed across all matching Pods
- **Service discovery**: Other Pods find services via DNS names
- **Internal only**: Not exposed externally, perfect for backend services


- Load balancer: 