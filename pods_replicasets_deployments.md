# Pods, ReplicaSets, and Deployments

## Overview

These three concepts form the foundation of Kubernetes resource management:
- **Pod**: Smallest deployable unit
- **ReplicaSet**: Ensures desired number of Pod replicas
- **Deployment**: Manages ReplicaSets and enables rolling updates

## Quick Reference

| Resource | Purpose | Use Case | Key Command |
|----------|---------|----------|------------|
| **Pod** | Wrap containers | Testing, debugging | `kubectl run <name> --image=<image>` |
| **ReplicaSet** | Maintain replicas | Rarely used directly | `kubectl scale replicaset <name> --replicas=N` |
| **Deployment** | Manage updates | **Production** ⭐ | `kubectl set image deployment/<name> <container>=<image>` |

---

## Pods

### What is a Pod?

A **Pod** is the smallest deployable object in Kubernetes. It's a wrapper around one or more containers.

**Key characteristics**:
- Contains one or more containers (usually one)
- Containers share network namespace (same IP address)
- Ephemeral (can be destroyed and recreated)
- Typically managed by higher-level objects (ReplicaSets, Deployments)

### Pod YAML Structure

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  labels:
    app: web
    env: production
spec:
  containers:
  - name: container-name
    image: nginx:latest
    ports:
    - containerPort: 80
```

### Required Fields

| Field | Purpose |
|-------|---------|
| `apiVersion: v1` | API version for Pod objects |
| `kind: Pod` | Type of Kubernetes resource |
| `metadata.name` | Name of the Pod |
| `metadata.labels` | Key-value pairs for identifying Pods |
| `spec.containers[0].name` | Container name within the Pod |
| `spec.containers[0].image` | Docker image to run |

### Creating a Pod

**Imperative method** (quick):
```bash
kubectl run nginx --image=nginx
```

**Declarative method** (best practice):
```bash
kubectl create -f pod-definition.yaml
```

### Viewing Pod Details

```bash
kubectl get pods                    # List all pods
kubectl describe pod <pod-name>     # Detailed info
kubectl logs <pod-name>             # View logs
kubectl exec -it <pod-name> -- sh   # Shell access
```

---

## ReplicaSets

### What is a ReplicaSet?

A **ReplicaSet** ensures that a specified number of Pod replicas are running at all times. If a Pod crashes, the ReplicaSet automatically creates a new one.

**Key characteristics**:
- Maintains desired number of Pod replicas
- Self-healing (creates new Pods if others fail)
- Uses labels/selectors to identify Pods
- Low-level resource (rarely used directly)

### Why ReplicaSets?

**Without ReplicaSet**: If a Pod crashes, it's gone forever.
```
Pod → Crashes → Pod is deleted → Application down ❌
```

**With ReplicaSet**: Failed Pods are automatically replaced.
```
Pod → Crashes → ReplicaSet detects it → Creates new Pod ✅
```

### ReplicaSet YAML Structure

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-replicaset
spec:
  replicas: 3                    # Desired number of Pods
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
```

### How ReplicaSets Work

1. **Selector**: Identifies which Pods to manage
   ```yaml
   selector:
     matchLabels:
       app: nginx    # Manages all Pods with this label
   ```

2. **Replicas**: Desired number of running Pods
   ```yaml
   replicas: 3       # Always maintain 3 Pods
   ```

3. **Template**: Blueprint for creating new Pods
   ```yaml
   template:         # Same as Pod spec
     metadata:
       labels:
         app: nginx
     spec:
       containers:
       - name: nginx
         image: nginx:latest
   ```

### ReplicaSet Example

Create 3 nginx Pods that self-heal:
```bash
kubectl create -f replicaset-definition.yaml
kubectl get replicasets
kubectl describe replicaset <name>
```

### ReplicaSet Selector (Label Matching)

The **selector** is crucial for ReplicaSets to identify which Pods to manage:

```yaml
selector:
  matchLabels:
    app: nginx           # Only manage Pods with this label
    env: production      # Multiple labels must ALL match
```

**How it works**:
- ReplicaSet searches for Pods matching ALL labels in `matchLabels`
- Any Pod with matching labels is adopted and managed
- Useful for grouping Pods by application, environment, or tier

**Scale with the scale command**:
```bash
kubectl scale replicaset nginx-replicaset --replicas=5
# Changes desired replicas from 3 to 5
```

### ReplicaSet Limitations

- Manual updates required (no rolling updates)
- Requires manual deletion and recreation for image changes
- Doesn't support declarative updates like Deployments
- Not typically used directly (use Deployments instead)

---

## Deployments

### What is a Deployment?

A **Deployment** is a high-level resource that manages ReplicaSets and Pods. It enables:
- **Rolling updates**: Update without downtime
- **Rollbacks**: Revert to previous versions instantly
- **Scaling**: Scale Pods up/down on demand
- **Pause/Resume**: Pause updates, make changes, then resume
- **Revision history**: Track all changes made to your deployment

**Key characteristics**:
- Manages ReplicaSets automatically
- Supports zero-downtime updates
- Complete revision history for rollbacks
- Most commonly used in production Kubernetes clusters
- Monitors and tracks all changes to your application

### Deployment Architecture

```
Deployment
  ↓
ReplicaSet (version 1)
  ├─ Pod 1
  ├─ Pod 2
  └─ Pod 3
```

When you update the image, Deployment creates a new ReplicaSet:
```
Deployment
  ├─ ReplicaSet (version 1) → 0 Pods (old version)
  └─ ReplicaSet (version 2) → 3 Pods (new version)
```

### Deployment YAML Structure

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1           # Max new Pods during update
      maxUnavailable: 1     # Max Pods that can be down
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14
        ports:
        - containerPort: 80
```

### Deployment vs ReplicaSet

| Feature | ReplicaSet | Deployment |
|---------|-----------|-----------|
| **Purpose** | Maintain pod replicas | Manage updates and rollbacks |
| **Self-healing** | ✅ Yes | ✅ Yes |
| **Rolling updates** | ❌ No | ✅ Yes |
| **Rollbacks** | ❌ No | ✅ Yes |
| **Version history** | ❌ No | ✅ Yes |
| **Production use** | Rarely | ✅ Always |

### Creating a Deployment

```bash
kubectl create deployment nginx --image=nginx:latest
```

Or using YAML:
```bash
kubectl create -f deployment-definition.yaml
```

### Managing Deployments

```bash
kubectl get deployments                           # List deployments
kubectl describe deployment <name>                # Details
kubectl get replicasets                           # See ReplicaSets (created automatically)
kubectl get pods                                  # See all Pods
kubectl scale deployment <name> --replicas=5      # Scale to 5 replicas
kubectl set image deployment/<name> <container>=<image>  # Update image
kubectl rollout status deployment/<name>          # Check update progress
kubectl rollout undo deployment/<name>            # Rollback to previous version
kubectl rollout history deployment/<name>         # View version history
```

### Rolling Update Example

**Original state**: 3 Pods running nginx:1.14
```bash
kubectl set image deployment/nginx-deployment nginx=nginx:1.15
```

**Update process** (RollingUpdate strategy):
1. Create 1 new Pod with nginx:1.15
2. Kill 1 old Pod
3. Create 1 new Pod with nginx:1.15
4. Kill 1 old Pod
5. Continue until all Pods are updated

**Result**: Zero-downtime update from 1.14 → 1.15

### Update Strategies

#### RollingUpdate (Default)
```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1              # Max 1 extra Pod during update
    maxUnavailable: 1        # Max 1 Pod can be down
```
✅ Zero downtime, gradual update

#### Recreate
```yaml
strategy:
  type: Recreate
```
❌ Kill all Pods, then create new ones (downtime)

### Advanced Deployment Operations

#### Pause and Resume Updates

Sometimes you want to pause a deployment to make gradual changes:

```bash
kubectl rollout pause deployment/nginx-deployment
# Now make changes - they won't immediately affect running Pods
```

Resume the deployment to apply all pending changes:
```bash
kubectl rollout resume deployment/nginx-deployment
```

**Use case**: Making multiple changes and rolling them out together

#### Scale Replicas

Change the number of running Pods:
```bash
kubectl scale deployment <name> --replicas=5
```

**Note**: ReplicaSets also support scaling:
```bash
kubectl scale replicaset <name> --replicas=5
```

#### Monitor Changes

View the complete revision history:
```bash
kubectl rollout history deployment/nginx-deployment
```

View details of a specific revision:
```bash
kubectl rollout history deployment/nginx-deployment --revision=2
```

#### Rollback to Previous Version

Instantly revert to the previous version:
```bash
kubectl rollout undo deployment/nginx-deployment
```

Rollback to a specific revision:
```bash
kubectl rollout undo deployment/nginx-deployment --to-revision=2
```

**Why this matters**: If a new image has bugs, rollback takes seconds with zero downtime

### Deployment Revisions and Version Control

Deployments automatically track **revision history**, allowing you to manage different versions:

#### How Revisions Work

Each time you update a Deployment (change image, replicas, etc.), Kubernetes creates a new revision:

```
Revision 1: nginx:1.14 with 3 replicas
    ↓ (update image)
Revision 2: nginx:1.15 with 3 replicas
    ↓ (scale to 5 replicas)
Revision 3: nginx:1.15 with 5 replicas
```

#### View Revision History

```bash
kubectl rollout history deployment/nginx-deployment
```

Output:
```
REVISION  CHANGE-CAUSE
1         kubectl set image deployment/nginx-deployment nginx=nginx:1.14
2         kubectl set image deployment/nginx-deployment nginx=nginx:1.15
3         kubectl scale deployment/nginx-deployment --replicas=5
```

#### Why Revisions Matter

1. **Quick rollbacks**: If a new version has bugs, revert instantly to any previous revision
2. **No server shutdown**: Rolling updates mean zero downtime; revisions ensure safe fallback
3. **Version tracking**: Audit trail of all changes made to your deployment
4. **Safe updates**: Always test new versions; revert quickly if issues arise

#### Update Strategies Comparison

| Strategy | Downtime | Speed | Use Case |
|----------|----------|-------|----------|
| **RollingUpdate** | ❌ No (zero downtime) | Medium (gradual) | Production (default) |
| **Recreate** | ✅ Yes (all Pods down) | Fast (instant) | Dev/Testing only |

**Recommendation**: Always use **RollingUpdate** in production to maintain availability

---

## Common Scenarios and Solutions

### Scenario 1: My Deployment Update Failed - How Do I Rollback?

**Problem**: Deployed a new image, but application crashes
**Solution**:
```bash
kubectl rollout undo deployment/my-app
# Instantly reverts to previous working version
```

### Scenario 2: How Do I Check if My Deployment is Updating?

**Command**:
```bash
kubectl rollout status deployment/my-app
# Shows real-time update progress
# Output: Waiting for rollout to finish...
#         3 out of 3 new replicas have been updated...
```

### Scenario 3: I Need to Update Multiple Things - How Do I Batch Changes?

**Solution**: Use pause/resume
```bash
kubectl rollout pause deployment/my-app
kubectl set image deployment/my-app app=myapp:v2
kubectl set env deployment/my-app ENV=production
kubectl rollout resume deployment/my-app
# All changes roll out together in one update
```

### Scenario 4: How Do I See What Changed Between Revisions?

**Command**:
```bash
kubectl rollout history deployment/my-app --revision=2
# Shows exact configuration of that revision
```

### Scenario 5: How Do I Scale My Application Quickly?

**For Deployment**:
```bash
kubectl scale deployment my-app --replicas=10
```

**For ReplicaSet**:
```bash
kubectl scale replicaset my-app-rs --replicas=10
```

---

## Troubleshooting Checklist

| Issue | Check | Solution |
|-------|-------|----------|
| Pods won't start | `kubectl describe pod <name>` | Check image name, resources, node capacity |
| Deployment stuck | `kubectl rollout status deploy/<name>` | Check replica status, resource requests |
| Updates too slow | Review strategy settings | Increase `maxSurge` for faster rollout |
| Can't rollback | `kubectl rollout history deploy/<name>` | Ensure revision history is available |
| Pods keep crashing | `kubectl logs <pod-name>` | Check application logs for errors |

---

## Practical Comparison

### Scenario: Deploying a Web App

**Using Pods alone** ❌
```bash
kubectl create -f web-pod.yaml
# If pod crashes → app is down
```

**Using ReplicaSet** ✅ (better)
```bash
kubectl create -f web-replicaset.yaml
# If pod crashes → ReplicaSet creates new one
```

**Using Deployment** ✅✅ (best)
```bash
kubectl create -f web-deployment.yaml
# Self-heals + supports updates + rollbacks
```

### Complete Deployment Example

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  labels:
    app: web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
  template:
    metadata:
      labels:
        app: web
      spec:
        containers:
        - name: web-container
          image: myapp:v1.0
          ports:
          - containerPort: 8080
          env:
          - name: LOG_LEVEL
            value: "INFO"
          resources:
            requests:
              memory: "64Mi"
              cpu: "100m"
            limits:
              memory: "128Mi"
              cpu: "200m"
```

---

## Key Takeaways

### Resource Hierarchy
1. **Pod is the smallest unit**: Every container runs inside a Pod
2. **ReplicaSets ensure availability**: Automatically replaces failed Pods using label selectors
3. **Deployments manage ReplicaSets**: Adds rolling updates, rollbacks, and version history

### Core Concepts
4. **YAML is declarative**: Define desired state, Kubernetes makes it happen
5. **Selector/labels are critical**: Used to match Pods to ReplicaSets and Deployments
6. **Scaling**: Use `kubectl scale` to change replicas for both ReplicaSets and Deployments

### Deployment-Specific Features
7. **Rolling updates = zero downtime**: Gradually update Pods without stopping the app
8. **Rollbacks are instant**: `kubectl rollout undo` reverts to any previous version
9. **Pause and resume**: Use `kubectl rollout pause/resume` to stage multiple changes
10. **Monitor all changes**: `kubectl rollout history` tracks every deployment revision

### Production Best Practices
11. **Always use Deployments**: Not Pods or ReplicaSets directly for production
12. **Pin image versions**: Use specific tags (nginx:1.27) not "latest"
13. **Revision history**: Enables fast rollbacks when issues arise
14. **Test updates**: Always test new versions before full deployment rollout

### Revisions and Updates
15. **Keep track of versions**: Revision history lets you track all deployment changes
16. **Zero-downtime updates**: Use rolling updates to deploy new versions without server shutdown
17. **RollingUpdate vs Recreate**: RollingUpdate (default) is for production; Recreate causes downtime