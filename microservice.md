# Microservices Architecture in Kubernetes

## Overview

Microservices architecture breaks down applications into small, independent services that work together. Each service is:
- **Self-contained**: Has its own codebase and deployment
- **Loosely coupled**: Communicates via well-defined APIs
- **Independently scalable**: Can scale based on its own needs
- **Technology diverse**: Each service can use different technologies

---

## Why Microservices?

### Problems with Monolithic Architecture

```
Monolith Problem:
┌─────────────────────────────────────┐
│         Single Application          │
│ ├─ Voting Logic                    │
│ ├─ Worker Processing               │
│ ├─ Database Operations             │
│ ├─ Result Display                  │
│ └─ Cache Management                │
└─────────────────────────────────────┘
- Hard to scale individual components
- One failure brings down entire app
- Difficult to deploy new features
- Technology lock-in
```

### Microservices Solution

```
Microservices Solution:
┌──────────────┐  ┌──────────┐  ┌────────────┐
│ Voting App   │  │ Worker   │  │ Result App │
│ (Python)     │  │ (.NET)   │  │ (Node.js)  │
└──────────────┘  └──────────┘  └────────────┘
       │                │               │
       └────────┬───────┴───────┬───────┘
                │               │
           ┌────▼────────┐  ┌──▼───────┐
           │    Redis    │  │PostgreSQL │
           │   (Cache)   │  │ (Database)│
           └─────────────┘  └───────────┘

Benefits:
✅ Scale each service independently
✅ Fault isolation (one failure doesn't affect others)
✅ Deploy services independently
✅ Use best technology for each service
```

---

## Real-World Example: Voting Application

### Architecture Overview

A complete voting application with multiple microservices:

```
┌─────────────────────────────────────────────────────────────┐
│                    Users/Clients                            │
└────────────┬──────────────────────────────────────┬─────────┘
             │                                      │
      ┌──────▼────────┐                    ┌───────▼────────┐
      │  Frontend     │                    │  Result App    │
      │  (React)      │                    │  (Node.js)     │
      │  Port: 3000   │                    │  Port: 5001    │
      └──────┬────────┘                    └───────┬────────┘
             │                                      │
             │      ClusterIP Service               │
             │    voting-service:80                │
             │                                      │
      ┌──────▼──────────────────────────────────────▼──┐
      │  Backend API (Python)                           │
      │  Voting Logic                                   │
      │  voting-app:5000                               │
      └─────────────┬──────────────────────────────┬────┘
                    │                              │
       ┌────────────▼────────┐        ┌───────────▼──────────┐
       │  Worker Service     │        │  PostgreSQL Database │
       │  (.NET)             │        │  postgres:5432       │
       │  worker:8080        │        └──────────────────────┘
       └────────────────────┘
                    │
       ┌────────────▼────────┐
       │  Redis Cache        │
       │  redis:6379         │
       └─────────────────────┘
```

### Components Breakdown

#### 1. Voting App (Python Backend)
**Purpose**: Handle voting requests and business logic

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: voting-app
  labels:
    app: voting-app
spec:
  containers:
  - name: voting-app
    image: voting-app:latest
    ports:
    - containerPort: 5000
    env:
    - name: REDIS_HOST
      value: redis  # Service name for auto-discovery
    - name: REDIS_PORT
      value: "6379"
```

**Key Features**:
- Listens on port 5000
- Connects to Redis for caching votes
- Connects to PostgreSQL to store results
- Exposes HTTP API for voting

#### 2. Worker Service (.NET)
**Purpose**: Process votes and update database

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: worker
  labels:
    app: worker
spec:
  containers:
  - name: worker
    image: worker:latest
    env:
    - name: REDIS_HOST
      value: redis      # Service discovery
    - name: DB_HOST
      value: postgres   # Service discovery
    - name: DB_PORT
      value: "5432"
    - name: DB_NAME
      value: "votes"
    - name: DB_USER
      value: "postgres"
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: password
```

**Key Features**:
- Listens to Redis queue for vote messages
- Processes votes asynchronously
- Updates PostgreSQL database
- No direct user interaction

#### 3. Result App (Node.js)
**Purpose**: Display voting results

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: result-app
  labels:
    app: result-app
spec:
  containers:
  - name: result-app
    image: result-app:latest
    ports:
    - containerPort: 5001
    env:
    - name: DB_HOST
      value: postgres   # Service discovery
    - name: DB_PORT
      value: "5432"
    - name: DB_USER
      value: "postgres"
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: password
```

**Key Features**:
- Listens on port 5001
- Queries PostgreSQL for results
- Real-time updates (WebSockets)
- Displays live voting results

#### 4. Redis (Cache)
**Purpose**: Queue votes and cache data

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: redis
  labels:
    app: redis
spec:
  containers:
  - name: redis
    image: redis:latest
    ports:
    - containerPort: 6379
```

**Key Features**:
- In-memory data store
- Queue votes for processing
- Fast vote counting
- Session caching

#### 5. PostgreSQL (Database)
**Purpose**: Persistent data storage

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: postgres
  labels:
    app: postgres-db
spec:
  containers:
  - name: postgres
    image: postgres:latest
    ports:
    - containerPort: 5432
    env:
    - name: POSTGRES_DB
      value: "votes"
    - name: POSTGRES_USER
      value: "postgres"
    - name: POSTGRES_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: password
    volumeMounts:
    - name: postgres-storage
      mountPath: /var/lib/postgresql/data
  volumes:
  - name: postgres-storage
    emptyDir: {}
```

**Key Features**:
- Persistent vote storage
- ACID transactions
- Query interface for result app

---

## Service Communication Setup

### Services Expose the Applications

```yaml
# Voting App Service (Internal - ClusterIP)
apiVersion: v1
kind: Service
metadata:
  name: voting-service
spec:
  type: ClusterIP
  selector:
    app: voting-app
  ports:
  - port: 80
    targetPort: 5000

# Result App Service (External - NodePort or LoadBalancer)
---
apiVersion: v1
kind: Service
metadata:
  name: result-service
spec:
  type: LoadBalancer
  selector:
    app: result-app
  ports:
  - port: 80
    targetPort: 5001
    nodePort: 30005

# Redis Service (Internal - ClusterIP)
---
apiVersion: v1
kind: Service
metadata:
  name: redis
spec:
  type: ClusterIP
  selector:
    app: redis
  ports:
  - port: 6379
    targetPort: 6379

# PostgreSQL Service (Internal - ClusterIP)
---
apiVersion: v1
kind: Service
metadata:
  name: postgres
spec:
  type: ClusterIP
  selector:
    app: postgres-db
  ports:
  - port: 5432
    targetPort: 5432
```

### Inter-Service Communication

Services communicate using **service names** for automatic DNS resolution:

**Voting App → Redis**:
```python
import redis

# Using service name for discovery
redis_client = redis.Redis(
    host='redis',           # Service name (auto-discovered by DNS)
    port=6379,
    decode_responses=True
)

# Vote is pushed to queue
redis_client.rpush('votes', vote_data)
```

**Voting App → PostgreSQL**:
```python
import psycopg2

# Database connection via service
connection = psycopg2.connect(
    host="postgres",        # Service name
    port=5432,
    database="votes",
    user=os.getenv("DB_USER"),
    password=os.getenv("DB_PASSWORD")
)
```

**Worker → PostgreSQL**:
```csharp
using System;
using Npgsql;

// Connection string using service name
string connString = $"Host=postgres;Port=5432;Username=postgres;Password={dbPassword};Database=votes";
using var conn = new NpgsqlConnection(connString);
conn.Open();
```

**Worker → Redis**:
```csharp
using StackExchange.Redis;

// Connect using service name
var redis = ConnectionMultiplexer.Connect("redis:6379");
var db = redis.GetDatabase();

// Get votes from queue
var vote = db.ListLeftPop("votes");
```

**Result App → PostgreSQL**:
```javascript
const pool = new Pool({
  host: process.env.DB_HOST || 'postgres',
  port: 5432,
  database: 'votes',
  user: 'postgres',
  password: process.env.DB_PASSWORD
});
```

---

## Environment Configuration & Secrets

### Environment Variables

Create ConfigMap for non-sensitive configuration:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  REDIS_HOST: "redis"
  REDIS_PORT: "6379"
  DB_HOST: "postgres"
  DB_PORT: "5432"
  DB_NAME: "votes"
  DB_USER: "postgres"
```

### Secrets for Credentials

Create Secret for sensitive data:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
stringData:
  password: "SecurePassword123!"
```

### Using ConfigMap and Secrets in Pods

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: voting-app
spec:
  containers:
  - name: voting-app
    image: voting-app:latest
    envFrom:
    - configMapRef:
        name: app-config  # Load all ConfigMap values
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: password
```

---

## Deployment Architecture

### Full Application Deployment

```yaml
# Voting App Deployment
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: voting-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: voting-app
  template:
    metadata:
      labels:
        app: voting-app
    spec:
      containers:
      - name: voting-app
        image: voting-app:v1.0
        ports:
        - containerPort: 5000
        envFrom:
        - configMapRef:
            name: app-config
        env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password
        resources:
          requests:
            memory: "256Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "500m"

# Worker Deployment
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: worker
spec:
  replicas: 2
  selector:
    matchLabels:
      app: worker
  template:
    metadata:
      labels:
        app: worker
    spec:
      containers:
      - name: worker
        image: worker:v1.0
        envFrom:
        - configMapRef:
            name: app-config
        env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password
        resources:
          requests:
            memory: "128Mi"
            cpu: "50m"
          limits:
            memory: "256Mi"
            cpu: "200m"

# Result App Deployment
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: result-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: result-app
  template:
    metadata:
      labels:
        app: result-app
    spec:
      containers:
      - name: result-app
        image: result-app:v1.0
        ports:
        - containerPort: 5001
        envFrom:
        - configMapRef:
            name: app-config
        env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password
        resources:
          requests:
            memory: "256Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "500m"
```

---

## Best Practices for Microservices

### 1. Service Isolation
- Each service should have its own database
- Communicate via APIs, not shared databases
- Independent deployment cycles

### 2. Environment Configuration
- Use ConfigMaps for non-sensitive config
- Use Secrets for passwords and credentials
- Match environment variable names with code expectations

### 3. Resource Management
- Define requests and limits for each container
- Prevents resource starvation
- Enables proper scheduling

### 4. Service Discovery
- Use service names instead of hardcoded IPs
- Kubernetes DNS resolves automatically
- Makes services location-independent

### 5. Logging & Monitoring
```bash
# Check service logs
kubectl logs deployment/voting-app

# Monitor service traffic
kubectl describe service voting-service

# Check inter-service connectivity
kubectl exec -it <pod-name> -- sh
# curl http://redis:6379
# curl http://postgres:5432
```

### 6. Health Checks
```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 5000
  initialDelaySeconds: 10
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /ready
    port: 5000
  initialDelaySeconds: 5
  periodSeconds: 5
```

---

## Troubleshooting Microservices

### Service Can't Connect to Another Service

```bash
# 1. Check if service exists
kubectl get services

# 2. Check service endpoints
kubectl get endpoints voting-service

# 3. Test connectivity from a Pod
kubectl exec -it <pod-name> -- sh
# curl http://voting-service:80
# ping redis
```

### Environment Variables Not Set

```bash
# Check if ConfigMap exists
kubectl get configmap

# View ConfigMap content
kubectl describe configmap app-config

# Check Pod environment
kubectl exec <pod-name> -- env | grep DB_
```

### Credentials Not Working

```bash
# Check if Secret exists
kubectl get secrets

# Verify secret content (shows only key names)
kubectl describe secret db-credentials

# Decode secret value (base64)
kubectl get secret db-credentials -o jsonpath='{.data.password}' | base64 -d
```

### Service Won't Start

```bash
# Check Pod status
kubectl get pods

# Get detailed status
kubectl describe pod <pod-name>

# Check logs
kubectl logs <pod-name>

# Check recent events
kubectl get events --sort-by='.lastTimestamp'
```

---

## Key Takeaways

1. **Microservices break down monoliths**: Each service has a specific responsibility
2. **Service independence**: Services scale, deploy, and fail independently
3. **Service discovery via DNS**: Use service names, not hardcoded IPs
4. **Services expose applications**: Use appropriate service types (ClusterIP, NodePort, LoadBalancer)
5. **Environment configuration**: Use ConfigMaps for non-sensitive data
6. **Secrets for credentials**: Store passwords and sensitive data securely
7. **Inter-service communication**: Services communicate via Kubernetes DNS
8. **Deployments manage services**: Use Deployments for managing multiple replicas
9. **Resource limits prevent starvation**: Define requests and limits for each service
10. **Monitoring is critical**: Log, monitor, and trace inter-service communication
