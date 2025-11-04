# Services & Networking

## Overview

Kubernetes Services provide stable networking and service discovery for Pods. Since Pods are ephemeral and can be recreated with different IPs, Services provide a consistent way to access them.

## The Problem Services Solve

```
Without Services:
┌────────┐  ┌────────┐  ┌────────┐
│ Pod 1  │  │ Pod 2  │  │ Pod 3  │
│10.0.1.5│  │10.0.1.6│  │10.0.1.7│
└────────┘  └────────┘  └────────┘
     ↓ Pod dies, IP changes ↓
┌────────┐  ┌────────┐  ┌────────┐
│ Pod 1  │  │ Pod 4  │  │ Pod 3  │
│10.0.1.5│  │10.0.2.8│  │10.0.1.7│  ❌ Clients can't track IPs
└────────┘  └────────┘  └────────┘

With Services:
              ┌─────────────┐
              │   Service   │
              │ 10.96.0.100 │  ✅ Stable IP + DNS
              └─────────────┘
                     │
        ┌────────────┼────────────┐
        ▼            ▼            ▼
   ┌────────┐  ┌────────┐  ┌────────┐
   │ Pod 1  │  │ Pod 2  │  │ Pod 3  │
   │(any IP)│  │(any IP)│  │(any IP)│
   └────────┘  └────────┘  └────────┘
```

---

## Service Types

### 1. ClusterIP (Default)

**Purpose:** Exposes service on cluster-internal IP. Only reachable from within cluster.

**Use Cases:**

- Internal microservice communication
- Database access within cluster
- Internal APIs

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: ClusterIP # Default, can be omitted
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80 # Service port
      targetPort: 8080 # Container port
```

**Diagram:**

```
┌───────────────────────────────────┐
│         Kubernetes Cluster        │
│                                   │
│  ┌─────────────────────────────┐ │
│  │  ClusterIP Service          │ │
│  │  10.96.0.100:80            │ │
│  └─────────────────────────────┘ │
│              │                    │
│     ┌────────┴────────┐         │
│     ▼                 ▼          │
│  ┌─────┐          ┌─────┐       │
│  │Pod 1│          │Pod 2│       │
│  │:8080│          │:8080│       │
│  └─────┘          └─────┘       │
│                                   │
└───────────────────────────────────┘
```

**Commands:**

```bash
# Create ClusterIP service
kubectl expose deployment my-app --port=80 --target-port=8080

# Or apply YAML
kubectl apply -f service.yaml

# Access from another pod
kubectl run test --image=busybox --rm -it --restart=Never -- \
  wget -qO- http://my-service
```

---

### 2. NodePort

**Purpose:** Exposes service on each Node's IP at a static port (30000-32767).

**Use Cases:**

- Development/testing access from outside cluster
- When LoadBalancer not available
- Direct node access needed

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: NodePort
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80 # Service port
      targetPort: 8080 # Container port
      nodePort: 30080 # Node port (optional, auto-assigned if omitted)
```

**Diagram:**

```
External Access
      │
      ▼
┌──────────────────────────────────┐
│   Node IP:30080                  │
├──────────────────────────────────┤
│                                  │
│  ┌────────────────────────────┐ │
│  │  NodePort Service          │ │
│  │  10.96.0.100:80           │ │
│  └────────────────────────────┘ │
│              │                   │
│     ┌────────┴────────┐        │
│     ▼                 ▼         │
│  ┌─────┐          ┌─────┐      │
│  │Pod 1│          │Pod 2│      │
│  │:8080│          │:8080│      │
│  └─────┘          └─────┘      │
│                                  │
└──────────────────────────────────┘
```

**Access:**

```bash
# Get node port
kubectl get service my-service

# Access from outside cluster
curl http://<NODE_IP>:30080

# On Minikube
minikube service my-service --url
```

---

### 3. LoadBalancer

**Purpose:** Exposes service externally using cloud provider's load balancer.

**Use Cases:**

- Production external access
- Cloud environments (AWS, GCP, Azure)
- Automatic external IP assignment

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: LoadBalancer
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
```

**Diagram:**

```
        Internet
           │
           ▼
  ┌──────────────────┐
  │  Load Balancer   │  ← Created by cloud provider
  │  External IP     │
  └──────────────────┘
           │
┌──────────┴─────────────────────┐
│      Kubernetes Cluster        │
│                                 │
│  ┌──────────────────────────┐ │
│  │  LoadBalancer Service    │ │
│  │  10.96.0.100:80         │ │
│  └──────────────────────────┘ │
│              │                  │
│     ┌────────┴────────┐       │
│     ▼                 ▼        │
│  ┌─────┐          ┌─────┐     │
│  │Pod 1│          │Pod 2│     │
│  │:8080│          │:8080│     │
│  └─────┘          └─────┘     │
│                                 │
└─────────────────────────────────┘
```

**Get external IP:**

```bash
kubectl get service my-service

# Wait for EXTERNAL-IP (may take a few minutes)
NAME         TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)
my-service   LoadBalancer   10.96.0.100     203.0.113.42    80:30080/TCP
```

**Note:** On local clusters (Minikube, Kind), LoadBalancer stays in "Pending" without cloud provider. Use `minikube tunnel` or MetalLB for testing.

---

### 4. ExternalName

**Purpose:** Maps service to external DNS name (CNAME record).

**Use Cases:**

- Access external services via Kubernetes DNS
- Service migration (gradual move to K8s)
- Database hosted outside cluster

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-db
spec:
  type: ExternalName
  externalName: database.example.com
```

**Usage:**

```bash
# Pods can access as: external-db.default.svc.cluster.local
# Which resolves to: database.example.com
```

---

## Service Discovery

### DNS-Based Discovery

Every Service gets a DNS name automatically:

**Format:** `<service-name>.<namespace>.svc.cluster.local`

**Examples:**

```bash
# Same namespace
curl http://my-service

# Different namespace
curl http://my-service.production

# Fully qualified
curl http://my-service.production.svc.cluster.local
```

### Environment Variables

Kubernetes injects service info as environment variables:

```bash
# Format:
<SERVICE_NAME>_SERVICE_HOST=10.96.0.100
<SERVICE_NAME>_SERVICE_PORT=80
```

**Example in Pod:**

```bash
kubectl exec my-pod -- env | grep MY_SERVICE
# Output:
MY_SERVICE_SERVICE_HOST=10.96.0.100
MY_SERVICE_SERVICE_PORT=80
```

---

## Endpoints

Services use **Endpoints** to track Pod IPs that match the selector.

```bash
# View endpoints
kubectl get endpoints my-service

# Detailed view
kubectl describe endpoints my-service
```

**Endpoint Object:**

```yaml
apiVersion: v1
kind: Endpoints
metadata:
  name: my-service
subsets:
  - addresses:
      - ip: 10.0.1.5
      - ip: 10.0.1.6
    ports:
      - port: 8080
```

---

## Headless Services

**Purpose:** Service without ClusterIP, returns Pod IPs directly (for StatefulSets, service discovery).

```yaml
apiVersion: v1
kind: Service
metadata:
  name: headless-service
spec:
  clusterIP: None # Makes it headless
  selector:
    app: my-app
  ports:
    - port: 80
```

**DNS Returns:**

- Normal Service: Single IP (ClusterIP)
- Headless Service: All Pod IPs

**Use Cases:**

- StatefulSet pod discovery
- Direct pod-to-pod communication
- Custom load balancing

---

## Ingress

**Purpose:** HTTP/HTTPS routing to services based on rules. Single entry point for multiple services.

### Why Ingress?

**Without Ingress:**

```
app1.com → LoadBalancer 1 → Service 1
app2.com → LoadBalancer 2 → Service 2  ❌ Multiple load balancers = $$$
app3.com → LoadBalancer 3 → Service 3
```

**With Ingress:**

```
app1.com ┐
app2.com ├→ Single LoadBalancer → Ingress Controller → Services ✅
app3.com ┘
```

### Ingress Resource

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
    - host: app1.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: service1
                port:
                  number: 80

    - host: app2.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: service2
                port:
                  number: 80
```

### Ingress with TLS

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
spec:
  tls:
    - hosts:
        - app.example.com
      secretName: tls-secret
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-service
                port:
                  number: 80
```

### Path-Based Routing

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: path-ingress
spec:
  rules:
    - host: example.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 80

          - path: /web
            pathType: Prefix
            backend:
              service:
                name: web-service
                port:
                  number: 80
```

### Ingress Controllers

Popular options:

- **NGINX Ingress Controller** (most common)
- **Traefik**
- **HAProxy**
- **Contour**
- **Ambassador**

**Install NGINX Ingress Controller:**

```bash
# On Minikube
minikube addons enable ingress

# On other clusters
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.1/deploy/static/provider/cloud/deploy.yaml

# Verify
kubectl get pods -n ingress-nginx
```

---

## Network Policies

**Purpose:** Control traffic flow between Pods (firewall rules for Pods).

### Default Behavior

By default, all Pods can communicate with all Pods.

### Deny All Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
  namespace: production
spec:
  podSelector: {} # Apply to all pods
  policyTypes:
    - Ingress
```

### Allow Specific Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 8080
```

### Allow from Specific Namespace

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-namespace
spec:
  podSelector:
    matchLabels:
      app: database
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              name: production
```

### Egress Policy

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-egress-to-api
spec:
  podSelector:
    matchLabels:
      app: frontend
  policyTypes:
    - Egress
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: api
      ports:
        - protocol: TCP
          port: 8080
```

**Note:** Network Policies require a CNI plugin that supports them (Calico, Cilium, Weave Net).

---

## Service Mesh (Advanced)

For complex microservice communication, consider a **Service Mesh**:

- **Istio** - Full-featured, powerful
- **Linkerd** - Lightweight, easy to use
- **Consul** - HashiCorp ecosystem

**Service Mesh Features:**

- Mutual TLS between services
- Advanced traffic routing (canary, blue-green)
- Observability (metrics, tracing)
- Circuit breaking & retries
- Rate limiting

---

## Practical Examples

### Example 1: Multi-Tier Application

```yaml
# Database Service (ClusterIP)
apiVersion: v1
kind: Service
metadata:
  name: database
spec:
  selector:
    app: db
  ports:
    - port: 5432
---
# API Service (ClusterIP)
apiVersion: v1
kind: Service
metadata:
  name: api
spec:
  selector:
    app: api
  ports:
    - port: 8080
---
# Frontend Service (LoadBalancer)
apiVersion: v1
kind: Service
metadata:
  name: frontend
spec:
  type: LoadBalancer
  selector:
    app: frontend
  ports:
    - port: 80
      targetPort: 3000
```

### Example 2: Ingress Setup

```yaml
# Deployment
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
        - name: nginx
          image: nginx
          ports:
            - containerPort: 80
---
# Service
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  selector:
    app: web
  ports:
    - port: 80
---
# Ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
spec:
  rules:
    - host: myapp.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web-service
                port:
                  number: 80
```

**Test:**

```bash
# Apply all
kubectl apply -f app.yaml

# Add to /etc/hosts (for local testing)
echo "$(minikube ip) myapp.local" | sudo tee -a /etc/hosts

# Access
curl http://myapp.local
```

---

## Troubleshooting Services

```bash
# Check service
kubectl get service my-service
kubectl describe service my-service

# Check endpoints (should list Pod IPs)
kubectl get endpoints my-service

# Test from within cluster
kubectl run test --image=busybox --rm -it --restart=Never -- \
  wget -qO- http://my-service

# Check DNS
kubectl run test --image=busybox --rm -it --restart=Never -- \
  nslookup my-service

# View service logs (via pods)
kubectl logs -l app=my-app

# Port forward for local testing
kubectl port-forward service/my-service 8080:80
```

**Common Issues:**

1. **Endpoints empty** - Check pod labels match service selector
2. **Connection refused** - Check targetPort matches container port
3. **DNS not working** - Check CoreDNS pods in kube-system
4. **External IP pending** - LoadBalancer needs cloud provider support

---

## Key Takeaways

1. **Services** provide stable networking for ephemeral Pods
2. **ClusterIP** for internal, **NodePort** for dev, **LoadBalancer** for prod
3. **Ingress** provides HTTP/HTTPS routing to multiple services
4. **Network Policies** control traffic between Pods
5. **DNS** enables service discovery by name
6. Use **labels** and **selectors** to connect Services to Pods

---

## Next Steps

👉 **[08-storage-volumes.md](./08-storage-volumes.md)** - Learn about persistent storage

## Additional Resources

- [Service Documentation](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Ingress Documentation](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- [Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [NGINX Ingress Controller](https://kubernetes.github.io/ingress-nginx/)
