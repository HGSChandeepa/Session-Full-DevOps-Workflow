# Best Practices & Production Readiness

## Overview

This guide consolidates best practices for running Kubernetes in production, covering architecture, security, operations, cost optimization, and disaster recovery.

---

## Architecture Best Practices

### 1. Namespace Organization

```yaml
# Organize by environment
namespaces:
  - development
  - staging
  - production

# Or by team/project
namespaces:
  - team-platform
  - team-data
  - team-mobile

# System namespaces (don't modify)
  - kube-system
  - kube-public
  - kube-node-lease
```

**Namespace Template:**

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    environment: production
    team: platform
    cost-center: engineering
    pod-security.kubernetes.io/enforce: restricted
  annotations:
    description: "Production workloads"
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    requests.cpu: "100"
    requests.memory: 200Gi
    limits.cpu: "200"
    limits.memory: 400Gi
    persistentvolumeclaims: "50"
    services.loadbalancers: "5"
---
apiVersion: v1
kind: LimitRange
metadata:
  name: production-limits
  namespace: production
spec:
  limits:
    - max:
        cpu: "4"
        memory: 8Gi
      min:
        cpu: 100m
        memory: 128Mi
      default:
        cpu: 500m
        memory: 512Mi
      defaultRequest:
        cpu: 200m
        memory: 256Mi
      type: Container
```

### 2. High Availability

**Control Plane:**

```
┌────────────────────────────────────┐
│   3+ Control Plane Nodes           │
│   ┌──────┐  ┌──────┐  ┌──────┐   │
│   │Master│  │Master│  │Master│   │
│   │  1   │  │  2   │  │  3   │   │
│   └──────┘  └──────┘  └──────┘   │
│        │         │         │       │
│        └────────┬─────────┘       │
│                 │                  │
│   ┌─────────────▼──────────────┐  │
│   │    Load Balancer           │  │
│   └────────────────────────────┘  │
└────────────────────────────────────┘
```

**etcd Cluster:**

```yaml
# Odd number of etcd nodes (3, 5, or 7)
# Spread across availability zones
# Regular backups (automated)
```

**Worker Nodes:**

```yaml
# Multiple nodes across AZs
# Autoscaling enabled
# At least 3 nodes minimum
# Node anti-affinity for critical apps
```

### 3. Pod Anti-Affinity

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 3
  template:
    spec:
      # Spread pods across nodes
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - web-app
            topologyKey: kubernetes.io/hostname

        # Prefer different AZs
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - web-app
              topologyKey: topology.kubernetes.io/zone
```

### 4. Resource Management

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: production-pod
spec:
  containers:
    - name: app
      image: myapp:1.0

      # ALWAYS set requests and limits
      resources:
        requests:
          cpu: 200m # Scheduling decision
          memory: 256Mi
        limits:
          cpu: 500m # Hard limit
          memory: 512Mi

      # Quality of Service (QoS)
      # - Guaranteed: requests == limits
      # - Burstable: requests < limits
      # - BestEffort: no requests/limits (avoid!)
```

**QoS Classes:**

```yaml
# Guaranteed (highest priority)
resources:
  requests:
    cpu: 500m
    memory: 512Mi
  limits:
    cpu: 500m
    memory: 512Mi

# Burstable (medium priority)
resources:
  requests:
    cpu: 200m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi

# BestEffort (lowest priority - avoid!)
# No resources defined
```

---

## Security Best Practices

### 1. Pod Security Standards

```yaml
# Apply at namespace level
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

### 2. Secure Pod Configuration

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 10000
    fsGroup: 10000
    seccompProfile:
      type: RuntimeDefault

  containers:
    - name: app
      image: myapp:1.0

      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        runAsNonRoot: true
        capabilities:
          drop:
            - ALL

      # Use volume for writable dirs
      volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: cache
          mountPath: /app/cache

  volumes:
    - name: tmp
      emptyDir: {}
    - name: cache
      emptyDir: {}
```

### 3. Network Policies

```yaml
# Default deny all
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
---
# Allow specific traffic
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: app-network-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: myapp
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              role: frontend
      ports:
        - protocol: TCP
          port: 8080
  egress:
    - to:
        - podSelector:
            matchLabels:
              role: database
      ports:
        - protocol: TCP
          port: 5432
    # Allow DNS
    - to:
        - namespaceSelector:
            matchLabels:
              name: kube-system
      ports:
        - protocol: UDP
          port: 53
```

### 4. RBAC

```yaml
# Least privilege principle
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: app-deployer
  namespace: production
rules:
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "update", "patch"]
  - apiGroups: [""]
    resources: ["pods", "pods/log"]
    verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-deployer-binding
  namespace: production
subjects:
  - kind: ServiceAccount
    name: ci-deployer
    namespace: production
roleRef:
  kind: Role
  name: app-deployer
  apiGroup: rbac.authorization.k8s.io
```

---

## Reliability Best Practices

### 1. Health Checks

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: reliable-app
spec:
  template:
    spec:
      containers:
        - name: app
          image: myapp:1.0

          # Startup probe (for slow-starting apps)
          startupProbe:
            httpGet:
              path: /startup
              port: 8080
            failureThreshold: 30
            periodSeconds: 10

          # Liveness probe (detect deadlocks)
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 3

          # Readiness probe (traffic control)
          readinessProbe:
            httpGet:
              path: /ready
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5
            timeoutSeconds: 3
            successThreshold: 1
            failureThreshold: 3
```

### 2. Graceful Shutdown

```yaml
spec:
  template:
    spec:
      terminationGracePeriodSeconds: 60

      containers:
        - name: app
          image: myapp:1.0

          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 15"]
```

**Application Code (Python example):**

```python
import signal
import sys

def signal_handler(sig, frame):
    print('Shutting down gracefully...')
    # Close database connections
    # Finish in-flight requests
    # Cleanup resources
    sys.exit(0)

signal.signal(signal.SIGTERM, signal_handler)
```

### 3. PodDisruptionBudget

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: app-pdb
spec:
  minAvailable: 2 # or maxUnavailable: 1
  selector:
    matchLabels:
      app: myapp
```

### 4. HorizontalPodAutoscaler

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 3
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 50
          periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
        - type: Percent
          value: 100
          periodSeconds: 30
        - type: Pods
          value: 4
          periodSeconds: 30
      selectPolicy: Max
```

---

## Observability Best Practices

### 1. Logging

```yaml
# Structured logging
spec:
  containers:
    - name: app
      image: myapp:1.0
      env:
        - name: LOG_FORMAT
          value: "json"
        - name: LOG_LEVEL
          value: "info"
```

**Application Code:**

```python
import json
import logging

logger = logging.getLogger(__name__)

def log_event(level, message, **kwargs):
    log_data = {
        "timestamp": datetime.utcnow().isoformat(),
        "level": level,
        "message": message,
        "service": "myapp",
        "version": "1.0.0",
        **kwargs
    }
    logger.log(getattr(logging, level), json.dumps(log_data))

# Usage
log_event("INFO", "User logged in", user_id=123, ip="1.2.3.4")
```

### 2. Metrics

```yaml
# Expose metrics endpoint
apiVersion: v1
kind: Service
metadata:
  name: app-metrics
  labels:
    app: myapp
spec:
  ports:
    - name: metrics
      port: 8080
  selector:
    app: myapp
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: app-metrics
spec:
  selector:
    matchLabels:
      app: myapp
  endpoints:
    - port: metrics
      interval: 30s
```

### 3. Alerts

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: production-alerts
spec:
  groups:
    - name: production.rules
      interval: 30s
      rules:
        - alert: HighErrorRate
          expr: |
            rate(http_requests_total{status=~"5.."}[5m]) > 0.05
          for: 5m
          labels:
            severity: critical
            team: platform
          annotations:
            summary: "High error rate detected"
            description: "Error rate is {{ $value | humanizePercentage }}"
            runbook_url: "https://wiki.company.com/runbooks/high-error-rate"

        - alert: PodCrashLooping
          expr: |
            rate(kube_pod_container_status_restarts_total[15m]) > 0
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "Pod {{ $labels.pod }} is crash looping"
```

---

## Cost Optimization

### 1. Resource Right-Sizing

```bash
# Analyze actual usage
kubectl top pods -n production --sort-by=cpu
kubectl top pods -n production --sort-by=memory

# Use VPA (Vertical Pod Autoscaler) for recommendations
kubectl describe vpa myapp-vpa
```

### 2. Cluster Autoscaling

```yaml
# Cloud provider autoscaling
# AWS EKS example:
apiVersion: autoscaling/v1
kind: NodeGroup
metadata:
  name: production
spec:
  minSize: 3
  maxSize: 20
  desiredCapacity: 5
  instanceTypes:
    - t3.large
    - t3.xlarge
```

### 3. Spot Instances

```yaml
# Use spot instances for non-critical workloads
nodeSelector:
  node.kubernetes.io/instance-type: spot

tolerations:
  - key: node.kubernetes.io/instance-type
    value: spot
    effect: NoSchedule
```

### 4. Resource Quotas

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-quota
  namespace: team-a
spec:
  hard:
    requests.cpu: "50"
    requests.memory: 100Gi
    limits.cpu: "100"
    limits.memory: 200Gi
    persistentvolumeclaims: "20"
```

---

## Backup & Disaster Recovery

### 1. etcd Backup

```bash
# Automated backup script
#!/bin/bash
BACKUP_DIR="/backups/etcd"
TIMESTAMP=$(date +%Y%m%d-%H%M%S)

ETCDCTL_API=3 etcdctl snapshot save \
  ${BACKUP_DIR}/etcd-snapshot-${TIMESTAMP}.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Upload to S3
aws s3 cp ${BACKUP_DIR}/etcd-snapshot-${TIMESTAMP}.db \
  s3://my-backups/etcd/

# Keep only last 7 days
find ${BACKUP_DIR} -name "etcd-snapshot-*.db" -mtime +7 -delete
```

**Run as CronJob:**

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: etcd-backup
  namespace: kube-system
spec:
  schedule: "0 2 * * *" # Daily at 2 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: backup
              image: k8s.gcr.io/etcd:3.5.0
              command: ["/backup.sh"]
              volumeMounts:
                - name: backup-script
                  mountPath: /backup.sh
                  subPath: backup.sh
          restartPolicy: OnFailure
          volumes:
            - name: backup-script
              configMap:
                name: etcd-backup-script
                defaultMode: 0755
```

### 2. Persistent Volume Backups

```bash
# Velero for cluster backups
helm repo add vmware-tanzu https://vmware-tanzu.github.io/helm-charts
helm install velero vmware-tanzu/velero \
  --namespace velero \
  --create-namespace \
  --set configuration.provider=aws \
  --set configuration.backupStorageLocation.bucket=my-backups \
  --set configuration.backupStorageLocation.config.region=us-east-1

# Create backup
velero backup create full-backup --include-namespaces production

# Schedule backups
velero schedule create daily-backup \
  --schedule="0 2 * * *" \
  --include-namespaces production \
  --ttl 168h  # 7 days
```

### 3. Disaster Recovery Plan

```yaml
DR_PLAN:
  1. Backup Strategy:
    - etcd: Daily automated backups
    - PVs: Snapshot-based backups
    - Cluster config: GitOps repository

  2. Recovery Procedures:
    - Document step-by-step restoration
    - Test recovery quarterly
    - RTO: 4 hours
    - RPO: 24 hours

  3. Multi-Region Setup:
    - Primary cluster: us-east-1
    - Secondary cluster: us-west-2
    - DNS failover configured

  4. Testing:
    - Monthly DR drills
    - Validate backup integrity
    - Update documentation
```

---

## Performance Best Practices

### 1. Image Optimization

```dockerfile
# Multi-stage build
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN npm run build

# Final stage
FROM node:18-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
USER node
EXPOSE 3000
CMD ["node", "dist/index.js"]
```

### 2. Caching

```yaml
# Use image pull policy wisely
spec:
  containers:
    - name: app
      image: myapp:1.0.0
      imagePullPolicy: IfNotPresent # Don't always pull
```

### 3. DNS Optimization

```yaml
# Configure DNS cache
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
          pods insecure
          fallthrough in-addr.arpa ip6.arpa
          ttl 30
        }
        cache 30
        loop
        reload
        loadbalance
    }
```

**Pod DNS Config:**

```yaml
spec:
  dnsConfig:
    options:
      - name: ndots
        value: "1" # Reduce DNS queries
```

---

## Documentation Best Practices

### 1. README Template

````markdown
# Application Name

## Overview

Brief description of the application

## Architecture

- Components diagram
- Dependencies
- Data flow

## Deployment

```bash
kubectl apply -f k8s/
```
````

## Configuration

Environment variables and ConfigMaps

## Monitoring

- Grafana dashboards
- Alert rules
- SLOs

## Runbooks

- [High Error Rate](runbooks/high-error-rate.md)
- [Pod Crash Loop](runbooks/pod-crash-loop.md)

## Contacts

- Team: platform@company.com
- On-call: pagerduty.com/platform

````

### 2. Manifest Comments

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  # Owner: Platform Team
  # Contact: platform@company.com
  # Documentation: https://wiki.company.com/myapp
  annotations:
    version: "1.0.0"
    change-cause: "Updated to latest security patches"
spec:
  # Replicas set based on load testing
  # Peak traffic: 10k req/s requires 5 pods
  replicas: 5
  # ... rest of config
````

---

## Deployment Checklist

### Pre-Production Checklist

- [ ] Resource requests and limits set
- [ ] Health checks configured (liveness, readiness)
- [ ] Security context defined (non-root, read-only)
- [ ] Network policies in place
- [ ] RBAC configured with least privilege
- [ ] Secrets encrypted at rest
- [ ] HPA configured for scalability
- [ ] PDB configured for availability
- [ ] Monitoring and alerts set up
- [ ] Logging configured (structured JSON)
- [ ] Backup strategy defined
- [ ] Disaster recovery tested
- [ ] Documentation complete
- [ ] Load tested
- [ ] Security scanned
- [ ] Cost estimated

---

## Key Takeaways

1. **Always set** resource requests and limits
2. **Implement** proper health checks
3. **Use** namespace isolation
4. **Apply** security best practices (non-root, read-only)
5. **Enable** HPA and PDB for reliability
6. **Configure** monitoring and alerting
7. **Backup** etcd and persistent volumes regularly
8. **Test** disaster recovery procedures
9. **Optimize** images and resource usage
10. **Document** everything thoroughly

---

## Production Readiness Score

Rate your cluster (0-10 for each):

- [ ] High Availability Setup
- [ ] Security Hardening
- [ ] Monitoring & Alerting
- [ ] Backup & DR
- [ ] Cost Optimization
- [ ] Performance Tuning
- [ ] Documentation
- [ ] Team Training

**Target: 80/80 for production-ready**

---

## Next Steps

👉 Review all previous guides and implement these best practices in your cluster

## Additional Resources

- [Kubernetes Production Best Practices](https://kubernetes.io/docs/setup/best-practices/)
- [Google SRE Book](https://sre.google/books/)
- [12 Factor App](https://12factor.net/)
- [CNCF Cloud Native Trail Map](https://github.com/cncf/trailmap)
- [Kubernetes the Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way)
