# Kubernetes Networking

## Overview

Kubernetes networking enables communication between Pods, Services, and external clients. It's designed to be flexible, scalable, and secure.

## Kubernetes Networking Model

### Core Principles

Kubernetes enforces a **flat network model** with these fundamental rules:

1. **Every Pod gets its own IP address**
   - Pods can communicate directly via IP
   - No need for port mapping between Pods
   - Simplifies application design

2. **All Pods can communicate with each other without NAT (Network Address Translation)**
   - Within the same node
   - Across different nodes
   - No special configuration needed

3. **All nodes can communicate with all Pods (and vice versa)**
   - Bidirectional communication
   - Without Network Address Translation (NAT)
   - No firewalls between nodes and Pods

4. **Pod sees its own IP as others see it**
   - No hidden NAT translation
   - IP remains consistent regardless of location
   - Simplifies debugging and troubleshooting

### Why No NAT?

**Traditional VM/Container Approach (WITH NAT)**:
```
Pod A (IP: 10.1.1.5) → NAT → External IP (192.168.1.10)
Pod B needs to know External IP, not internal IP
❌ Complicated, harder to debug
```

**Kubernetes Approach (WITHOUT NAT)**:
```
Pod A (IP: 10.1.1.5) → Pod B (IP: 10.2.2.8)
Direct communication with same IP
✅ Simple, transparent, easier to debug
```

---

## Network Architecture

### Layers of Kubernetes Networking

```
┌─────────────────────────────────────────────────┐
│         External Networks                       │
│  (Internet, corporate networks, etc.)           │
└────────────────┬────────────────────────────────┘
                 │
         ┌───────▼──────────┐
         │   LoadBalancer   │
         │   or Ingress     │
         └───────┬──────────┘
                 │
    ┌────────────┴────────────┐
    │                         │
┌───▼──────────┐      ┌──────▼─────┐
│    Node 1    │      │   Node 2   │
├──────────────┤      ├────────────┤
│ Pod A        │      │ Pod C      │
│ IP: 10.1.1.5 │      │ IP: 10.2.2.8
│              │      │            │
│ Pod B        │      │ Pod D      │
│ IP: 10.1.1.6 │      │ IP: 10.2.2.9
└──────────────┘      └────────────┘
```

### Network Components

1. **Pod Network**
   - Virtual network where Pods operate
   - Each Pod gets unique IP address
   - Pods on same node and different nodes can communicate

2. **Node Network**
   - Physical/virtual network connecting nodes
   - Each node has IP address for external communication
   - Minikube: Virtual network (192.168.x.x or similar)

3. **Service Network**
   - Virtual network layer on top of Pod network
   - Provides stable IP and DNS for accessing Pods
   - Handles load balancing

4. **Cluster DNS**
   - CoreDNS service in Kubernetes
   - Resolves service names to cluster IPs
   - Example: `my-service.default.svc.cluster.local`

---

## Networking Layers in Kubernetes

### Layer 1: Pod-to-Pod Communication (Same Node)

**Setup**: 2 Pods on the same node

```
Pod A (10.1.1.5) ──┐
                   ├─► Docker Bridge Network (veth)
Pod B (10.1.1.6) ──┘

Communication: Direct via bridge, no routing needed
Speed: Fast (no network hops)
```

**How it works**:
- Pods connected to docker0 bridge on the node
- Docker bridge routes traffic between Pods
- No external network involved

### Layer 2: Pod-to-Pod Communication (Different Nodes)

**Setup**: Pod on Node 1 talking to Pod on Node 2

```
Node 1                         Node 2
┌──────────────┐              ┌──────────────┐
│ Pod A        │              │ Pod C        │
│ 10.1.1.5     │              │ 10.2.2.8     │
└──────┬───────┘              └──────┬───────┘
       │                             │
   ┌───▼───────────────────────────┬───┐
   │   Cluster Network             │   │
   │   (CNI Plugin)                │   │
   └───────────────────────────────┴───┘

Flow: Pod A → Node 1 → Network → Node 2 → Pod C
```

**How it works**:
- Kubernetes uses **CNI (Container Network Interface)** plugins
- CNI manages inter-node routing
- Examples: Flannel, Calico, Weave

### Layer 3: Pod-to-Service Communication

**Setup**: Pod accessing application via Service

```
Pod A (10.1.1.5)
    │
    ▼
Service (10.96.0.1:80) ← ClusterIP
    │
    ├─► Pod B (10.1.1.6) ← Load balanced
    ├─► Pod C (10.2.2.8)
    └─► Pod D (10.2.2.9)
```

**How it works**:
- Service provides stable IP and DNS name
- kube-proxy handles load balancing (iptables/ipvs)
- No need to know individual Pod IPs

---

## Services: Exposing Applications

### What is a Service?

A **Service** is a Kubernetes abstraction that:
- Provides stable IP address for accessing Pods
- Distributes traffic across multiple Pods
- Eliminates need to track individual Pod IPs
- Provides DNS name (example: `my-app.default.svc.cluster.local`)

### Service Types

#### 1. ClusterIP (Default)

**Purpose**: Internal communication within the cluster

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
    - port: 80              # Service port
      targetPort: 8080      # Pod container port
```

**Access**:
- From other Pods: `web-service:80` or `web-service.default.svc.cluster.local:80`
- Not accessible from outside cluster

**Use case**: Backend services, internal APIs

#### 2. NodePort

**Purpose**: Expose service to external traffic via node IP

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
    - port: 80              # Service port
      targetPort: 8080      # Pod port
      nodePort: 30001       # Node port (30000-32767)
```

**Access**:
- External: `<node-ip>:30001`
- Example: `192.168.1.10:30001`

**Flow**:
```
External Client
    │
    ▼
<Node IP>:30001
    │
    ▼
Service IP:80
    │
    ▼
Pod:8080
```

**Use case**: Development, testing, exposing to external networks

#### 3. LoadBalancer

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

**Access**:
- External Load Balancer IP (auto-provisioned)
- Example: `203.0.113.45:80`

**How it works**:
- Cloud provider (AWS, GCP, Azure) provisions load balancer
- Load balancer routes to NodePort services
- Works automatically on cloud platforms

**Use case**: Production, public APIs, real load balancing

#### 4. ExternalName

**Purpose**: Map service to external DNS name

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-db
spec:
  type: ExternalName
  externalName: database.example.com
```

**Use case**: Connecting to external databases, services outside cluster

### Service Load Balancing

Services distribute traffic across Pods using **selectors**:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  selector:
    app: web          # Match Pods with this label
  ports:
    - port: 80
      targetPort: 8080
```

All Pods with `app: web` label are automatically added to load balancing pool.

---

## Network Plugins (CNI)

### What is CNI?

**Container Network Interface (CNI)** is a standard for managing container networking in Kubernetes.

CNI plugins handle:
- IP address assignment
- Inter-node routing
- Network policies
- Encapsulation/tunneling

### Popular CNI Plugins

| Plugin | Technology | Features |
|--------|-----------|----------|
| **Flannel** | VXLAN overlay | Simple, lightweight, good for learning |
| **Calico** | BGP, IP-in-IP | High performance, network policies |
| **Weave** | Custom protocol | Simple setup, encrypted communication |
| **Cilium** | eBPF, BGP | Advanced, high performance, L7 policies |

**Minikube default**: Often uses Flannel or docker bridge

---

## Minikube Network Details

### Minikube Virtual Network

```
Host Machine (192.168.1.100)
    │
    ├─► Minikube VM (192.168.49.2)
        │
        └─► Pods Network (10.244.0.0/16)
            ├─ Pod A: 10.244.0.2
            ├─ Pod B: 10.244.0.3
            └─ Pod C: 10.244.1.2
```

### Accessing Minikube Services

From **host machine**:
```bash
minikube service my-service --url
# Returns: http://192.168.49.2:30001
# Open in browser: Use this URL
```

From **within cluster**:
```bash
kubectl run debug --image=nginx -it -- sh
# Inside Pod: curl http://my-service:80
```

---

## Common Networking Scenarios

### Scenario 1: Pod-to-Pod Communication (Same Cluster)

```bash
# Get Pod IP
kubectl get pods -o wide
# NAME       IP           NODE
# pod-a      10.244.0.2   node1
# pod-b      10.244.0.3   node1

# From pod-a, access pod-b directly
kubectl exec -it pod-a -- sh
# curl http://10.244.0.3:8080
```

### Scenario 2: Accessing Service from Another Pod

```bash
# Create service
kubectl expose pod web-pod --port=80 --target-port=8080

# From another pod
kubectl exec -it debug-pod -- sh
# curl http://web-service:80
# Or with FQDN
# curl http://web-service.default.svc.cluster.local:80
```

### Scenario 3: Exposing Service to External Traffic

```bash
# Create NodePort service
kubectl expose deployment web --type=NodePort --port=80

# Get the node port
kubectl get service web
# Shows NodePort: 30001

# Access from host
curl http://<minikube-ip>:30001
# Example: curl http://192.168.49.2:30001
```

### Scenario 4: Service Discovery with DNS

```bash
# Kubernetes automatically creates DNS records
# Format: <service-name>.<namespace>.svc.cluster.local

# Example: Access web-service from pod
kubectl run debug --image=nginx -it -- sh
# curl http://web-service.default.svc.cluster.local
# Shorter: curl http://web-service (within same namespace)
```

---

## Troubleshooting Network Issues

### Check Pod Connectivity

```bash
# Verify Pod IP
kubectl get pods -o wide

# Ping another Pod
kubectl exec -it pod-a -- ping 10.244.0.3

# Check service endpoints
kubectl get endpoints web-service

# Test service from within cluster
kubectl run debug --image=nginx -it -- sh
# curl http://web-service:80
```

### Debug Network Policies

```bash
# List network policies
kubectl get networkpolicies

# Describe specific policy
kubectl describe networkpolicy <policy-name>
```

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Pods can't reach each other | Network plugin not installed | Install CNI plugin (Flannel, Calico, etc.) |
| Service endpoints empty | Pod selector doesn't match | Verify labels with `kubectl get pods --show-labels` |
| External access fails | NodePort not open | Check firewall, verify port range 30000-32767 |
| DNS resolution fails | CoreDNS down | Check CoreDNS Pod: `kubectl get pods -n kube-system` |
| High latency | Inter-node traffic | Consider network plugin performance, check node connectivity |

---

## Key Kubernetes Networking Principles

### Network Principles (Kubernetes Requirements)

1. **No NAT Model**: Pods communicate directly without address translation
2. **Flat Network**: All Pods see each other's IPs as they actually are
3. **IP-per-Pod**: Each Pod has unique, persistent IP within cluster
4. **No Port Mapping**: Containers expose ports directly (no mapping needed)

### Best Practices

1. **Use Services**: Don't access Pods directly, use Services for stable endpoints
2. **Label Pods**: Use clear labels for service selection and organization
3. **Limit with Network Policies**: Restrict traffic between namespaces/Pods
4. **Monitor DNS**: Ensure CoreDNS is running for service discovery
5. **Test Connectivity**: Use `kubectl exec` with curl/ping to debug

---

## Quick Reference

### Common Networking Commands

```bash
# Pod Information
kubectl get pods -o wide                        # See Pod IPs
kubectl describe pod <name>                     # Pod network details

# Service Information
kubectl get services                            # List services
kubectl describe service <name>                 # Service details
kubectl get endpoints <service>                 # Service endpoints (Pods)

# Network Policies
kubectl get networkpolicies                     # List policies
kubectl describe networkpolicy <name>           # Policy details

# Debugging
kubectl exec -it <pod> -- sh                    # Access Pod shell
kubectl logs <pod>                              # Check logs
kubectl port-forward <pod> 8080:8080            # Forward local port to Pod
```

### Service Type Comparison

| Type | Internal | External | Use Case |
|------|----------|----------|----------|
| **ClusterIP** | ✅ Yes | ❌ No | Internal services |
| **NodePort** | ✅ Yes | ✅ Yes (via node) | Dev/testing |
| **LoadBalancer** | ✅ Yes | ✅ Yes (via LB) | Production |
| **ExternalName** | ✅ Yes | N/A | External services |

---

## Key Takeaways

1. **Flat network model**: Kubernetes uses no-NAT networking for simplicity
2. **Pod-to-Pod communication**: Direct via Pod IPs across nodes
3. **Services provide stability**: Use Services, not Pod IPs, for access
4. **Service types vary**: ClusterIP (internal), NodePort (external), LoadBalancer (cloud)
5. **CNI plugins manage routing**: Flannel, Calico handle inter-node communication
6. **DNS service discovery**: Services get DNS names automatically
7. **Minikube has virtual network**: Virtual bridge between host and Pods
8. **Troubleshoot with kubectl**: Use exec, logs, and endpoints for debugging
9. **Network Policies control traffic**: Restrict communication between resources
10. **Labels are critical**: Service selection depends on Pod labels
