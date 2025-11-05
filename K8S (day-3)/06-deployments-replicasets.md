# Deployments & ReplicaSets Deep Dive

## Overview

Deployments and ReplicaSets are the foundation of running stateless applications in Kubernetes. Understanding them deeply is crucial for managing applications in production.

## ReplicaSets

### What is a ReplicaSet?

A **ReplicaSet** ensures that a specified number of pod replicas are running at any given time. It's a controller that maintains the desired state of pod replicas.

### ReplicaSet Architecture

```
┌─────────────────────────────────────────────┐
│           ReplicaSet Controller              │
│                                              │
│  Desired State: 3 replicas                  │
│  Current State: ? replicas                  │
│                                              │
│  Control Loop:                              │
│  1. Watch for changes                       │
│  2. Compare desired vs actual               │
│  3. Create/Delete pods as needed            │
│  4. Update status                           │
└─────────────────────────────────────────────┘
                    │
        ┌───────────┼───────────┐
        ▼           ▼           ▼
    ┌──────┐    ┌──────┐    ┌──────┐
    │Pod 1 │    │Pod 2 │    │Pod 3 │
    │ app  │    │ app  │    │ app  │
    └──────┘    └──────┘    └──────┘
```

### ReplicaSet YAML

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: frontend-rs
  labels:
    app: frontend
    tier: frontend
spec:
  # Number of pod replicas
  replicas: 3

  # Label selector for pods
  selector:
    matchLabels:
      app: frontend

  # Pod template
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
        - name: nginx
          image: nginx:1.21
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 200m
              memory: 256Mi
```

### Key Components

**1. Replicas**

- Desired number of pod copies
- Controller maintains this count

**2. Selector**

- Identifies which pods to manage
- Must match template labels
- Supports `matchLabels` and `matchExpressions`

**3. Template**

- Pod specification
- Used to create new pods
- Must have labels matching selector

### Selector Types

**matchLabels (Equality-based):**

```yaml
selector:
  matchLabels:
    app: frontend
    env: production
```

**matchExpressions (Set-based):**

```yaml
selector:
  matchExpressions:
    - key: app
      operator: In
      values:
        - frontend
        - backend
    - key: env
      operator: NotIn
      values:
        - dev
```

**Operators:**

- `In`: Label value in specified set
- `NotIn`: Label value not in specified set
- `Exists`: Label key exists
- `DoesNotExist`: Label key doesn't exist

### ReplicaSet Operations

```bash
# Create ReplicaSet
kubectl apply -f replicaset.yaml

# List ReplicaSets
kubectl get replicasets
kubectl get rs  # Short form

# Describe ReplicaSet
kubectl describe rs frontend-rs

# Scale ReplicaSet
kubectl scale rs frontend-rs --replicas=5

# Delete ReplicaSet
kubectl delete rs frontend-rs

# Delete ReplicaSet but keep pods
kubectl delete rs frontend-rs --cascade=orphan
```

### Self-Healing Example

```bash
# Create ReplicaSet with 3 replicas
kubectl apply -f replicaset.yaml

# Watch pods
kubectl get pods -w

# In another terminal, delete a pod
kubectl delete pod <pod-name>

# Watch as ReplicaSet creates a new pod automatically
```

### Why Not Use ReplicaSets Directly?

❌ **Don't use ReplicaSets directly because:**

- No built-in update mechanism
- Can't rollback
- No deployment strategies
- Manual version management

✅ **Use Deployments instead** - they manage ReplicaSets for you with additional features.

---

## Deployments

### What is a Deployment?

A **Deployment** provides declarative updates for Pods and ReplicaSets. It's the recommended way to manage stateless applications.

### Deployment Architecture

```
┌─────────────────────────────────────────────────┐
│               Deployment                         │
│   (Manages ReplicaSets & Rolling Updates)       │
└─────────────────────────────────────────────────┘
                    │
        ┌───────────┴───────────┐
        ▼                       ▼
┌──────────────────┐    ┌──────────────────┐
│  ReplicaSet v1   │    │  ReplicaSet v2   │
│  (old version)   │    │  (new version)   │
│  replicas: 0     │    │  replicas: 3     │
└──────────────────┘    └──────────────────┘
                               │
                    ┌──────────┼──────────┐
                    ▼          ▼          ▼
                ┌──────┐  ┌──────┐  ┌──────┐
                │Pod 1 │  │Pod 2 │  │Pod 3 │
                │ v2   │  │ v2   │  │ v2   │
                └──────┘  └──────┘  └──────┘
```

### Basic Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
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
          image: nginx:1.21
          ports:
            - containerPort: 80
```

### Advanced Deployment Configuration

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: advanced-deployment
  labels:
    app: myapp
  annotations:
    kubernetes.io/change-cause: "Update to version 2.0"
spec:
  replicas: 5

  # Deployment strategy
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1 # Max pods unavailable during update
      maxSurge: 1 # Max extra pods during update

  # Minimum time pod should be ready
  minReadySeconds: 5

  # Number of old ReplicaSets to retain
  revisionHistoryLimit: 10

  # Time in seconds for pod to be ready
  progressDeadlineSeconds: 600

  selector:
    matchLabels:
      app: myapp

  template:
    metadata:
      labels:
        app: myapp
        version: "2.0"
    spec:
      containers:
        - name: app
          image: myapp:2.0
          ports:
            - containerPort: 8080

          # Resource management
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 512Mi

          # Liveness probe
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10

          # Readiness probe
          readinessProbe:
            httpGet:
              path: /ready
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5

          # Environment variables
          env:
            - name: ENV
              value: production
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: url
```

---

## Deployment Strategies

### 1. Rolling Update (Default)

Gradually replaces old pods with new ones.

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1 # Max 1 pod down at a time
      maxSurge: 1 # Max 1 extra pod during update
```

**Process:**

```
Initial:    [v1] [v1] [v1]
Step 1:     [v1] [v1] [v1] [v2]  (surge)
Step 2:     [v1] [v1] [v2]       (v1 terminated)
Step 3:     [v1] [v1] [v2] [v2]  (surge)
Step 4:     [v1] [v2] [v2]       (v1 terminated)
Step 5:     [v1] [v2] [v2] [v2]  (surge)
Final:      [v2] [v2] [v2]       (v1 terminated)
```

**Advantages:**

- Zero downtime
- Gradual rollout
- Can pause and resume
- Easy rollback

**Use When:**

- Production deployments
- Need zero downtime
- Want gradual rollout

### 2. Recreate

Terminates all old pods before creating new ones.

```yaml
spec:
  strategy:
    type: Recreate
```

**Process:**

```
Initial:    [v1] [v1] [v1]
            ↓ Delete all
During:     (no pods running)
            ↓ Create new
Final:      [v2] [v2] [v2]
```

**Advantages:**

- Simple
- Complete state refresh
- No version mixing

**Disadvantages:**

- Downtime during update
- All-or-nothing approach

**Use When:**

- Can tolerate downtime
- Need to avoid version mixing
- Database migrations required
- Development/testing environments

### 3. Blue-Green (Manual with Deployments)

Run two identical environments, switch traffic at once.

```yaml
# Blue deployment (current)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: blue
  template:
    metadata:
      labels:
        app: myapp
        version: blue
    spec:
      containers:
        - name: myapp
          image: myapp:1.0
---
# Green deployment (new)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: green
  template:
    metadata:
      labels:
        app: myapp
        version: green
    spec:
      containers:
        - name: myapp
          image: myapp:2.0
---
# Service (switch between blue and green)
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  selector:
    app: myapp
    version: blue # Change to 'green' to switch
  ports:
    - port: 80
```

**Switch Traffic:**

```bash
# Update service selector
kubectl patch service myapp-service -p '{"spec":{"selector":{"version":"green"}}}'
```

### 4. Canary (Manual with Deployments)

Deploy new version to subset of users first.

```yaml
# Stable deployment (90%)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-stable
spec:
  replicas: 9
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
        version: stable
    spec:
      containers:
        - name: myapp
          image: myapp:1.0
---
# Canary deployment (10%)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-canary
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
        version: canary
    spec:
      containers:
        - name: myapp
          image: myapp:2.0
---
# Single service for both
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: myapp # Matches both deployments
  ports:
    - port: 80
```

---

## Deployment Operations

### Creating Deployments

```bash
# Imperative
kubectl create deployment nginx --image=nginx --replicas=3

# Declarative
kubectl apply -f deployment.yaml

# With additional options
kubectl create deployment nginx \
  --image=nginx:1.21 \
  --replicas=3 \
  --port=80

# Generate YAML
kubectl create deployment nginx --image=nginx --dry-run=client -o yaml > deployment.yaml
```

### Viewing Deployments

```bash
# List deployments
kubectl get deployments
kubectl get deploy -o wide

# Watch deployments
kubectl get deployments -w

# Describe deployment
kubectl describe deployment nginx-deployment

# Get deployment YAML
kubectl get deployment nginx-deployment -o yaml

# View deployment history
kubectl rollout history deployment/nginx-deployment
```

### Updating Deployments

```bash
# Update image (triggers rolling update)
kubectl set image deployment/nginx-deployment nginx=nginx:1.22

# Update multiple containers
kubectl set image deployment/nginx-deployment \
  nginx=nginx:1.22 \
  sidecar=sidecar:2.0

# Edit deployment directly
kubectl edit deployment nginx-deployment

# Update from file
kubectl apply -f deployment-updated.yaml

# Patch deployment
kubectl patch deployment nginx-deployment -p '{"spec":{"replicas":5}}'

# Update with annotation (for change-cause)
kubectl set image deployment/nginx-deployment nginx=nginx:1.22 \
  --record
```

### Scaling Deployments

```bash
# Scale manually
kubectl scale deployment nginx-deployment --replicas=5

# Autoscale based on CPU
kubectl autoscale deployment nginx-deployment \
  --min=2 \
  --max=10 \
  --cpu-percent=80

# View autoscaler
kubectl get hpa

# Delete autoscaler
kubectl delete hpa nginx-deployment
```

### Rolling Updates

```bash
# Check rollout status
kubectl rollout status deployment/nginx-deployment

# Watch rollout progress
kubectl rollout status deployment/nginx-deployment -w

# Pause rollout
kubectl rollout pause deployment/nginx-deployment

# Make changes while paused
kubectl set image deployment/nginx-deployment nginx=nginx:1.22

# Resume rollout
kubectl rollout resume deployment/nginx-deployment

# Restart deployment (recreate all pods)
kubectl rollout restart deployment/nginx-deployment
```

### Rollback

```bash
# View rollout history
kubectl rollout history deployment/nginx-deployment

# View specific revision
kubectl rollout history deployment/nginx-deployment --revision=2

# Rollback to previous version
kubectl rollout undo deployment/nginx-deployment

# Rollback to specific revision
kubectl rollout undo deployment/nginx-deployment --to-revision=2

# Check status after rollback
kubectl rollout status deployment/nginx-deployment
```

---

## Health Checks

### Liveness Probe

Checks if container is running. Restarts container if fails.

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 30 # Wait before first check
  periodSeconds: 10 # Check every 10 seconds
  timeoutSeconds: 5 # Timeout after 5 seconds
  failureThreshold: 3 # Restart after 3 failures
```

**Types:**

```yaml
# HTTP GET
livenessProbe:
  httpGet:
    path: /health
    port: 8080
    httpHeaders:
    - name: Custom-Header
      value: Awesome

# TCP Socket
livenessProbe:
  tcpSocket:
    port: 8080

# Exec command
livenessProbe:
  exec:
    command:
    - cat
    - /tmp/healthy
```

### Readiness Probe

Checks if container is ready to serve traffic. Removes from service if fails.

```yaml
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
  failureThreshold: 3
```

**Liveness vs Readiness:**

| Probe     | Failure Action      | Use Case                         |
| --------- | ------------------- | -------------------------------- |
| Liveness  | Restart container   | Detect deadlocks, hung processes |
| Readiness | Remove from service | Wait for dependencies, warmup    |

### Startup Probe

Checks if application has started. Disables liveness/readiness during startup.

```yaml
startupProbe:
  httpGet:
    path: /startup
    port: 8080
  initialDelaySeconds: 0
  periodSeconds: 10
  failureThreshold: 30 # 30 * 10 = 300s max startup time
```

**Complete Example:**

```yaml
spec:
  containers:
    - name: app
      image: myapp:1.0

      # Startup probe (runs first)
      startupProbe:
        httpGet:
          path: /startup
          port: 8080
        failureThreshold: 30
        periodSeconds: 10

      # Liveness probe (after startup)
      livenessProbe:
        httpGet:
          path: /health
          port: 8080
        initialDelaySeconds: 10
        periodSeconds: 10

      # Readiness probe (after startup)
      readinessProbe:
        httpGet:
          path: /ready
          port: 8080
        initialDelaySeconds: 5
        periodSeconds: 5
```

---

## Resource Management

```yaml
spec:
  containers:
    - name: app
      image: myapp:1.0
      resources:
        # Minimum resources (for scheduling)
        requests:
          cpu: 100m # 0.1 CPU cores
          memory: 128Mi # 128 megabytes

        # Maximum resources (enforced)
        limits:
          cpu: 500m # 0.5 CPU cores
          memory: 512Mi # 512 megabytes
```

**Resource Units:**

- CPU: millicores (m) - 1000m = 1 core
- Memory: Mi (mebibytes), Gi (gibibytes)

**Best Practices:**

- Always set requests for predictable scheduling
- Set limits to prevent resource exhaustion
- requests ≤ limits
- Monitor actual usage to tune values

---

## Labels and Selectors Best Practices

```yaml
metadata:
  labels:
    # Recommended labels
    app.kubernetes.io/name: myapp
    app.kubernetes.io/instance: myapp-prod
    app.kubernetes.io/version: "1.0.0"
    app.kubernetes.io/component: frontend
    app.kubernetes.io/part-of: myplatform
    app.kubernetes.io/managed-by: helm

    # Custom labels
    environment: production
    team: platform
    cost-center: engineering
```

---

## Practical Examples

### Example 1: Production Web App

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  labels:
    app: webapp
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
        - name: webapp
          image: mywebapp:1.0
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: 200m
              memory: 256Mi
            limits:
              cpu: 500m
              memory: 512Mi
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /ready
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5
          env:
            - name: ENV
              value: production
```

### Example 2: Canary Deployment Workflow

```bash
# 1. Deploy stable version
kubectl apply -f webapp-stable.yaml

# 2. Deploy canary (10% traffic)
kubectl apply -f webapp-canary.yaml

# 3. Monitor canary
kubectl logs -l version=canary -f

# 4. If good, promote canary
kubectl scale deployment webapp-canary --replicas=9
kubectl scale deployment webapp-stable --replicas=1

# 5. Fully switch
kubectl scale deployment webapp-canary --replicas=10
kubectl delete deployment webapp-stable
kubectl patch deployment webapp-canary -p '{"metadata":{"name":"webapp-stable"}}'
```

---

## Troubleshooting

### Common Issues

**1. ImagePullBackOff**

```bash
# Check events
kubectl describe pod <pod-name>

# Common causes:
# - Wrong image name
# - Missing image registry credentials
# - Image doesn't exist
```

**2. CrashLoopBackOff**

```bash
# Check logs
kubectl logs <pod-name>
kubectl logs <pod-name> --previous

# Common causes:
# - Application error
# - Missing configuration
# - Failed health checks
```

**3. Deployment Not Rolling Out**

```bash
# Check rollout status
kubectl rollout status deployment/<name>

# Check events
kubectl describe deployment <name>

# Check pod status
kubectl get pods -l app=<name>
```

### Debug Commands

```bash
# View detailed deployment info
kubectl describe deployment <name>

# Check ReplicaSets
kubectl get rs -l app=<name>

# View pod logs
kubectl logs -l app=<name> --tail=50

# Execute command in pod
kubectl exec -it <pod-name> -- sh

# View events
kubectl get events --sort-by='.lastTimestamp'

# Check resource usage
kubectl top pods -l app=<name>
```

---

## Best Practices

1. **Always use Deployments**, not bare Pods or ReplicaSets
2. **Set resource requests and limits**
3. **Implement health checks** (liveness, readiness, startup)
4. **Use rolling updates** with sensible maxUnavailable and maxSurge
5. **Keep revision history** (revisionHistoryLimit: 10)
6. **Use meaningful labels** for organization
7. **Test updates in staging** before production
8. **Monitor rollouts** and be ready to rollback
9. **Use `--record` flag** to track change causes
10. **Implement gradual rollouts** for critical services

---

## Key Takeaways

1. **ReplicaSets** maintain desired pod count, rarely used directly
2. **Deployments** manage ReplicaSets with rollout capabilities
3. **Rolling Update** is default strategy, zero downtime
4. **Health checks** are critical for reliability
5. **Resource management** prevents resource exhaustion
6. **Rollback** capability is essential for production
7. Always **monitor and validate** deployments

---

## Next Steps

👉 **[07-services-networking.md](./07-services-networking.md)** - Connect your deployments with Services

## Additional Resources

- [Deployment Documentation](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [ReplicaSet Documentation](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/)
- [Pod Lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/)
- [Configure Liveness, Readiness Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
