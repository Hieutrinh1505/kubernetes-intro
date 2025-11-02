# Kubernetes Concepts & Hands-On Setup

## Local Kubernetes Setup with Minikube

### What is Minikube?

**Minikube** is a tool that creates a local Kubernetes cluster on your machine. It's perfect for:
- Learning Kubernetes
- Local development and testing
- Running Kubernetes experiments without cloud costs

### Installation

Install the required tools using Homebrew (macOS):

```bash
brew install kubectl      # Kubernetes CLI
brew install minikube     # Local Kubernetes cluster
```

### Starting Your Local Cluster

```bash
minikube start --driver=docker
```

**What this does**:
- Creates a single-node Kubernetes cluster
- Uses Docker as the virtualization driver
- Installs and configures kubectl to connect to the cluster

**Verify installation**:
```bash
kubectl cluster-info      # Check cluster is running
kubectl get nodes         # Should show one node (minikube)
```

---

## Basic Kubernetes Workflow Example

### Step 1: Create a Deployment

A **Deployment** manages a set of identical pods and ensures they stay running.

```bash
kubectl create deployment hello-minikube --image=k8s.gcr.io/echoserver:1.10
```

**Output**: `deployment.apps/hello-minikube created`

**What this does**:
- Creates a deployment named "hello-minikube"
- Pulls the echoserver image
- Creates a pod running this container
- Manages the pod lifecycle (restarts if it crashes)

**Check the deployment**:
```bash
kubectl get deployments
kubectl get pods
```

### Step 2: Expose the Deployment as a Service

A **Service** provides networking to access your pods.

```bash
kubectl expose deployment hello-minikube --type=NodePort --port=8080
```

**What this does**:
- Creates a Service that exposes the deployment
- Type `NodePort` makes it accessible from outside the cluster
- Maps port 8080 inside the container

**Verify the service**:
```bash
kubectl get services
```

### Step 3: Access Your Application

```bash
minikube service hello-minikube --url
```

**Output**: Returns a URL like `http://192.168.49.2:30001`

Open this URL in your browser to see the running application!

### Step 4: Clean Up Resources

When you're done testing, delete the resources:

```bash
kubectl delete service hello-minikube       # Delete the service
kubectl delete deployment hello-minikube    # Delete the deployment (also deletes pods)
```

**Note**: Always delete services before deployments to ensure proper cleanup.

---

## Key Kubernetes Concepts

### Deployment
- **Purpose**: Manages the desired state of your application
- **Features**: Rolling updates, rollbacks, scaling
- **Use case**: "I want 3 replicas of my app running at all times"

### Pod
- **Definition**: Smallest deployable unit in Kubernetes
- **Contains**: One or more containers (usually one)
- **Lifecycle**: Managed by Deployments or other controllers
- **Ephemeral**: Pods are temporary; they can be destroyed and recreated

### Service
- **Purpose**: Provides stable networking for pods
- **Why needed**: Pods have changing IP addresses; Services provide a stable endpoint
- **Types**:
  - `ClusterIP` - Internal only (default)
  - `NodePort` - Accessible from outside via node IP
  - `LoadBalancer` - Cloud load balancer (AWS, GCP, Azure)

---

## Useful Commands Reference

### Cluster Management
```bash
minikube start              # Start local cluster
minikube stop               # Stop cluster
minikube delete             # Delete cluster
minikube status             # Check cluster status
minikube dashboard          # Open Kubernetes dashboard
```

### Deployment Commands
```bash
kubectl create deployment <name> --image=<image>    # Create deployment
kubectl get deployments                              # List deployments
kubectl describe deployment <name>                   # Deployment details
kubectl delete deployment <name>                     # Delete deployment
kubectl scale deployment <name> --replicas=3         # Scale to 3 pods
```

### Service Commands
```bash
kubectl expose deployment <name> --type=NodePort --port=<port>
kubectl get services                                 # List services
kubectl describe service <name>                      # Service details
kubectl delete service <name>                        # Delete service
```

### Pod Commands
```bash
kubectl get pods                    # List all pods
kubectl describe pod <name>         # Pod details
kubectl logs <pod-name>             # View pod logs
kubectl exec -it <pod-name> -- sh   # Shell into pod
kubectl delete pod <name>           # Delete pod (deployment will recreate it)
```

---

## Key Takeaways

1. **Minikube = Local Kubernetes**: Perfect for learning and development without cloud infrastructure
2. **Deployment manages Pods**: Ensures your application stays running with desired replicas
3. **Services provide networking**: Stable endpoint to access dynamically created pods
4. **Declarative management**: You specify what you want (3 replicas), Kubernetes makes it happen
5. **kubectl is your interface**: All interactions with Kubernetes go through kubectl commands
6. **Clean up resources**: Always delete services and deployments when done to free up resources

---

## Deep Dive: Pods

### What Are Pods?

**Pods** are the smallest and most fundamental deployable unit in Kubernetes. They act as a wrapper around one or more containers.

Key characteristics:
- **Smallest object**: You cannot deploy containers directly in Kubernetes; they must be wrapped in a Pod
- **Container encapsulation**: Pods encapsulate containers, providing them with networking and storage
- **Shared context**: Containers in the same Pod share the same network namespace (localhost), storage volumes, and IP address

### Pod Scaling Model

Kubernetes scales **horizontally** by adding/removing Pods, not by adding containers to existing Pods.

| Scaling Direction | How It Works |
|-------------------|--------------|
| **Scale Up** | Create new Pods (each with its own container) |
| **Scale Down** | Remove existing Pods |
| **NOT** | Add more containers to existing Pods |

**Example**: To handle 3x traffic, you create 3 Pods (not 1 Pod with 3 containers).

### Multi-Container Pods

While most Pods contain a **single container**, a Pod can have **multiple containers** when they need to work closely together.

**Use cases for multi-container Pods**:
- **Sidecar pattern**: Helper container supporting the main app (e.g., log forwarder)
- **Ambassador pattern**: Proxy container for external connections
- **Adapter pattern**: Standardizing output from the main container

**Example**:
```yaml
# Pod with main app + logging sidecar
Pod: web-app
  - Container 1: nginx (main application)
  - Container 2: log-shipper (sends logs to central system)
```

Containers in the same Pod:
- Share the same IP address
- Can communicate via `localhost`
- Share mounted volumes
- Are scheduled together on the same node

### Creating Pods with kubectl

#### Imperative Method (Quick & Simple)

**Example 1: Running nginx**
```bash
kubectl run nginx --image=nginx
```

**What this does**:
1. Pulls the `nginx` image from Docker Hub
2. Creates a Pod named "nginx"
3. Starts the container inside the Pod

**Verify**:
```bash
kubectl get pods
kubectl describe pod nginx
```

**Example 2: Running a custom service**
```bash
kubectl run service --image=service
```

**What this does**:
1. Creates a Pod named "service"
2. Pulls the image called "service" (could be from Docker Hub or your private registry)
3. Starts the container inside the Pod

**General syntax**:
```bash
kubectl run <pod-name> --image=<image-name>
```

- `<pod-name>`: Name you want to give your Pod
- `<image-name>`: Docker image to use (format: `[registry/]image[:tag]`)

#### Declarative Method (Production Best Practice)

Create a YAML file defining the Pod:

```yaml
# pod-definition.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
spec:
  containers:
  - name: nginx-container
    image: nginx
    ports:
    - containerPort: 80
```

Apply the configuration:
```bash
kubectl apply -f pod-definition.yaml
```

**Advantages of YAML**:
- Version control friendly
- Repeatable and auditable
- Easier to manage complex configurations

### Pod Lifecycle States

| State | Meaning |
|-------|---------|
| **Pending** | Pod accepted but not yet running (downloading images, scheduling) |
| **Running** | Pod bound to node, at least one container is running |
| **Succeeded** | All containers completed successfully and won't restart |
| **Failed** | All containers terminated, at least one failed |
| **Unknown** | Pod state cannot be determined (node communication issues) |

### Useful Pod Commands

```bash
kubectl run <name> --image=<image>          # Create a pod
kubectl get pods                             # List all pods
kubectl get pods -o wide                     # List with more details (IP, node)
kubectl describe pod <name>                  # Detailed pod information
kubectl logs <pod-name>                      # View container logs
kubectl logs <pod-name> -c <container-name> # Logs from specific container (multi-container pods)
kubectl exec -it <pod-name> -- sh            # Shell into the pod
kubectl delete pod <name>                    # Delete a pod
kubectl get pods --watch                     # Watch pods in real-time
```

### Key Takeaways: Pods

1. **Pods wrap containers**: Containers cannot run directly in Kubernetes; they must be in Pods
2. **Scale by adding Pods**: Kubernetes scaling = more Pods, not more containers per Pod
3. **Usually one container per Pod**: Unless containers need tight coupling (sidecar pattern)
4. **kubectl run is quick**: Creates a Pod instantly, but YAML is better for production
5. **Pods are ephemeral**: They can be destroyed and recreated; don't rely on Pod IPs staying the same