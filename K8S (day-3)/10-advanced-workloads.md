# Advanced Workloads

## Overview

Beyond Deployments, Kubernetes provides specialized workload controllers for different use cases: StatefulSets for stateful applications, DaemonSets for node-level operations, Jobs for batch processing, and CronJobs for scheduled tasks.

---

## StatefulSets

### What is a StatefulSet?

A **StatefulSet** manages stateful applications that require:

- Stable, unique network identifiers
- Stable, persistent storage
- Ordered, graceful deployment and scaling
- Ordered, automated rolling updates

### StatefulSet vs Deployment

| Feature              | Deployment         | StatefulSet                |
| -------------------- | ------------------ | -------------------------- |
| **Pod naming**       | Random suffix      | Ordered index (0, 1, 2...) |
| **Network identity** | Changes on restart | Stable (persistent)        |
| **Storage**          | Shared or none     | Dedicated per pod          |
| **Scaling**          | Parallel           | Sequential (ordered)       |
| **Updates**          | Parallel           | Sequential (ordered)       |
| **Use case**         | Stateless apps     | Stateful apps (databases)  |

### StatefulSet Architecture

```
StatefulSet: database (replicas: 3)
│
├── Pod: database-0
│   ├── Stable DNS: database-0.database.default.svc.cluster.local
│   └── PVC: data-database-0 (10Gi)
│
├── Pod: database-1
│   ├── Stable DNS: database-1.database.default.svc.cluster.local
│   └── PVC: data-database-1 (10Gi)
│
└── Pod: database-2
    ├── Stable DNS: database-2.database.default.svc.cluster.local
    └── PVC: data-database-2 (10Gi)
```

### Basic StatefulSet

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-headless
spec:
  clusterIP: None # Headless service
  selector:
    app: nginx
  ports:
    - port: 80
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: nginx
spec:
  serviceName: nginx-headless # Required
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
          volumeMounts:
            - name: www
              mountPath: /usr/share/nginx/html

  # Volume claim template
  volumeClaimTemplates:
    - metadata:
        name: www
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 1Gi
```

### Pod Identity

**DNS Names:**

```
<pod-name>.<service-name>.<namespace>.svc.cluster.local

Examples:
nginx-0.nginx-headless.default.svc.cluster.local
nginx-1.nginx-headless.default.svc.cluster.local
nginx-2.nginx-headless.default.svc.cluster.local
```

**Access specific pod:**

```bash
# From within cluster
curl nginx-0.nginx-headless.default.svc.cluster.local

# From same namespace
curl nginx-0.nginx-headless
```

### Scaling Behavior

**Scale Up (Sequential):**

```
Initial:  nginx-0
Scale +2: nginx-0 → nginx-1 → nginx-2
          (waits for each to be ready)
```

**Scale Down (Reverse Sequential):**

```
Initial:  nginx-0, nginx-1, nginx-2
Scale -2: nginx-2 → nginx-1 → nginx-0
          (deletes in reverse order)
```

```bash
# Scale StatefulSet
kubectl scale statefulset nginx --replicas=5

# Watch scaling
kubectl get pods -w -l app=nginx
```

### Update Strategies

**RollingUpdate (Default):**

```yaml
spec:
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 0 # Update pods >= this index
```

**OnDelete:**

```yaml
spec:
  updateStrategy:
    type: OnDelete # Update only when pod is manually deleted
```

**Partitioned Rollout:**

```yaml
spec:
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 2 # Only update pods 2 and higher
```

### Production Example: PostgreSQL

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
spec:
  clusterIP: None
  selector:
    app: postgres
  ports:
    - port: 5432
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-config
data:
  postgresql.conf: |
    max_connections = 100
    shared_buffers = 256MB
    effective_cache_size = 1GB
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:14
          ports:
            - containerPort: 5432
          env:
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: password
            - name: PGDATA
              value: /var/lib/postgresql/data/pgdata
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
            - name: config
              mountPath: /etc/postgresql
          livenessProbe:
            exec:
              command:
                - pg_isready
                - -U
                - postgres
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            exec:
              command:
                - pg_isready
                - -U
                - postgres
            initialDelaySeconds: 5
            periodSeconds: 5
      volumes:
        - name: config
          configMap:
            name: postgres-config

  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: fast-storage
        resources:
          requests:
            storage: 10Gi
```

### StatefulSet Operations

```bash
# Create StatefulSet
kubectl apply -f statefulset.yaml

# List StatefulSets
kubectl get statefulsets
kubectl get sts  # Short form

# Describe StatefulSet
kubectl describe statefulset postgres

# Scale StatefulSet
kubectl scale statefulset postgres --replicas=5

# Delete StatefulSet (keeps PVCs)
kubectl delete statefulset postgres

# Delete StatefulSet and PVCs
kubectl delete statefulset postgres
kubectl delete pvc -l app=postgres

# Update StatefulSet
kubectl apply -f statefulset-updated.yaml

# Check rollout status
kubectl rollout status statefulset postgres
```

---

## DaemonSets

### What is a DaemonSet?

A **DaemonSet** ensures that a copy of a pod runs on all (or selected) nodes. As nodes are added, pods are automatically added. As nodes are removed, pods are garbage collected.

### Use Cases

- **Node monitoring**: Prometheus Node Exporter, Datadog agent
- **Log collection**: Fluentd, Logstash, Filebeat
- **Storage daemons**: Ceph, GlusterFS
- **Network plugins**: CNI plugins like Calico, Weave
- **Security agents**: Falco, Twistlock

### DaemonSet Architecture

```
┌─────────────────────────────────────────┐
│           DaemonSet Controller          │
│  Ensures one pod per matching node     │
└─────────────────────────────────────────┘
                    │
        ┌───────────┼───────────┐
        ▼           ▼           ▼
    ┌──────┐    ┌──────┐    ┌──────┐
    │Node 1│    │Node 2│    │Node 3│
    │      │    │      │    │      │
    │ [DS] │    │ [DS] │    │ [DS] │  ← One pod per node
    └──────┘    └──────┘    └──────┘

When Node 4 is added:
    ┌──────┐    ┌──────┐    ┌──────┐    ┌──────┐
    │Node 1│    │Node 2│    │Node 3│    │Node 4│
    │ [DS] │    │ [DS] │    │ [DS] │    │ [DS] │ ← Auto-scheduled
    └──────┘    └──────┘    └──────┘    └──────┘
```

### Basic DaemonSet

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
  labels:
    app: fluentd
spec:
  selector:
    matchLabels:
      app: fluentd
  template:
    metadata:
      labels:
        app: fluentd
    spec:
      containers:
        - name: fluentd
          image: fluentd:v1.14
          resources:
            limits:
              memory: 200Mi
            requests:
              cpu: 100m
              memory: 200Mi
          volumeMounts:
            - name: varlog
              mountPath: /var/log
            - name: varlibdockercontainers
              mountPath: /var/lib/docker/containers
              readOnly: true
      volumes:
        - name: varlog
          hostPath:
            path: /var/log
        - name: varlibdockercontainers
          hostPath:
            path: /var/lib/docker/containers
```

### Node Selection

**Run on specific nodes with labels:**

```yaml
spec:
  template:
    spec:
      nodeSelector:
        disktype: ssd
        role: logging
```

**Run on specific nodes with affinity:**

```yaml
spec:
  template:
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: node-role.kubernetes.io/worker
                    operator: Exists
```

**Tolerate node taints:**

```yaml
spec:
  template:
    spec:
      tolerations:
        - key: node-role.kubernetes.io/master
          effect: NoSchedule
```

### Production Example: Node Monitoring

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: node-exporter
  template:
    metadata:
      labels:
        app: node-exporter
    spec:
      hostNetwork: true # Use host network
      hostPID: true # Use host PID namespace

      containers:
        - name: node-exporter
          image: prom/node-exporter:v1.5.0
          args:
            - --path.procfs=/host/proc
            - --path.sysfs=/host/sys
            - --path.rootfs=/host/root
            - --collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)
          ports:
            - containerPort: 9100
              hostPort: 9100
          resources:
            requests:
              cpu: 100m
              memory: 100Mi
            limits:
              cpu: 200m
              memory: 200Mi
          volumeMounts:
            - name: proc
              mountPath: /host/proc
              readOnly: true
            - name: sys
              mountPath: /host/sys
              readOnly: true
            - name: root
              mountPath: /host/root
              mountPropagation: HostToContainer
              readOnly: true

      volumes:
        - name: proc
          hostPath:
            path: /proc
        - name: sys
          hostPath:
            path: /sys
        - name: root
          hostPath:
            path: /

      tolerations:
        - effect: NoSchedule
          key: node-role.kubernetes.io/master
```

### DaemonSet Operations

```bash
# Create DaemonSet
kubectl apply -f daemonset.yaml

# List DaemonSets
kubectl get daemonsets
kubectl get ds  # Short form

# Describe DaemonSet
kubectl describe daemonset fluentd

# View DaemonSet pods
kubectl get pods -l app=fluentd -o wide

# Update DaemonSet
kubectl apply -f daemonset-updated.yaml

# Check rollout status
kubectl rollout status daemonset fluentd

# Delete DaemonSet
kubectl delete daemonset fluentd
```

---

## Jobs

### What is a Job?

A **Job** creates one or more pods and ensures a specified number complete successfully. When all pods complete, the Job is considered complete.

### Use Cases

- Database migrations
- Batch processing
- Data analysis
- Backup operations
- One-time administrative tasks

### Basic Job

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi-calculation
spec:
  template:
    spec:
      containers:
        - name: pi
          image: perl:5.34
          command: ["perl", "-Mbignum=bpi", "-wle", "print bpi(2000)"]
      restartPolicy: Never
  backoffLimit: 4 # Retry limit
```

### Job Patterns

#### Pattern 1: Single Completion

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: single-job
spec:
  completions: 1 # Run once
  parallelism: 1 # One at a time
  template:
    spec:
      containers:
        - name: task
          image: busybox
          command: ["sh", "-c", 'echo "Task completed"; sleep 30']
      restartPolicy: Never
```

#### Pattern 2: Parallel Jobs (Fixed Completion Count)

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: parallel-job
spec:
  completions: 10 # Complete 10 times
  parallelism: 3 # Run 3 at a time
  template:
    spec:
      containers:
        - name: worker
          image: busybox
          command: ["sh", "-c", 'echo "Processing item"; sleep 10']
      restartPolicy: Never
```

**Execution:**

```
Wave 1: [Pod1] [Pod2] [Pod3]  ← Running
Wave 2: [Pod4] [Pod5] [Pod6]  ← Waiting
Wave 3: [Pod7] [Pod8] [Pod9]  ← Waiting
Wave 4: [Pod10]                ← Waiting
```

#### Pattern 3: Work Queue (Parallel Processing)

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: work-queue
spec:
  parallelism: 5 # 5 workers
  completions: null # Run until queue is empty
  template:
    spec:
      containers:
        - name: worker
          image: myworker:1.0
          env:
            - name: QUEUE_URL
              value: "redis://queue:6379"
      restartPolicy: Never
```

### Job Configuration

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: advanced-job
spec:
  # Number of successful completions
  completions: 5

  # Number of pods running in parallel
  parallelism: 2

  # Retry limit for failed pods
  backoffLimit: 3

  # Job timeout (seconds)
  activeDeadlineSeconds: 600

  # Time to keep completed pods
  ttlSecondsAfterFinished: 100

  template:
    spec:
      containers:
        - name: task
          image: busybox
          command: ["sh", "-c", 'echo "Working..."; sleep 30']
      restartPolicy: Never # Or OnFailure
```

### Restart Policies

- `Never`: Create new pod on failure (increases pod count)
- `OnFailure`: Restart container in same pod (keeps pod count)

### Production Example: Database Backup

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: postgres-backup
spec:
  template:
    spec:
      containers:
        - name: backup
          image: postgres:14
          command:
            - /bin/bash
            - -c
            - |
              BACKUP_FILE="/backup/postgres-$(date +%Y%m%d-%H%M%S).sql"
              pg_dump -h postgres -U admin -d mydb > $BACKUP_FILE
              echo "Backup completed: $BACKUP_FILE"
          env:
            - name: PGPASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: password
          volumeMounts:
            - name: backup
              mountPath: /backup
      restartPolicy: Never
      volumes:
        - name: backup
          persistentVolumeClaim:
            claimName: backup-pvc
  backoffLimit: 3
```

### Job Operations

```bash
# Create Job
kubectl apply -f job.yaml

# List Jobs
kubectl get jobs

# Describe Job
kubectl describe job pi-calculation

# View Job pods
kubectl get pods -l job-name=pi-calculation

# View Job logs
kubectl logs -l job-name=pi-calculation

# Delete Job (and pods)
kubectl delete job pi-calculation

# Delete Job but keep pods
kubectl delete job pi-calculation --cascade=orphan
```

---

## CronJobs

### What is a CronJob?

A **CronJob** creates Jobs on a repeating schedule using cron format.

### Cron Schedule Format

```
┌───────────── minute (0 - 59)
│ ┌───────────── hour (0 - 23)
│ │ ┌───────────── day of month (1 - 31)
│ │ │ ┌───────────── month (1 - 12)
│ │ │ │ ┌───────────── day of week (0 - 6) (Sunday=0)
│ │ │ │ │
* * * * *
```

**Common Examples:**

```bash
*/5 * * * *     # Every 5 minutes
0 * * * *       # Every hour
0 0 * * *       # Every day at midnight
0 0 * * 0       # Every Sunday at midnight
0 0 1 * *       # First day of every month
0 9-17 * * 1-5  # 9 AM to 5 PM, Monday to Friday
```

### Basic CronJob

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: hello-cron
spec:
  schedule: "*/5 * * * *" # Every 5 minutes
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: hello
              image: busybox
              command: ["sh", "-c", 'echo "Hello from CronJob at $(date)"']
          restartPolicy: OnFailure
```

### Advanced CronJob

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: advanced-cronjob
spec:
  schedule: "0 2 * * *" # Every day at 2 AM

  # Timezone (K8s 1.25+)
  timeZone: "America/New_York"

  # Maximum time to start if missed
  startingDeadlineSeconds: 3600

  # Concurrency policy
  concurrencyPolicy: Forbid # Forbid, Allow, or Replace

  # Number of successful jobs to keep
  successfulJobsHistoryLimit: 3

  # Number of failed jobs to keep
  failedJobsHistoryLimit: 1

  # Suspend execution
  suspend: false

  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: task
              image: busybox
              command: ["sh", "-c", 'echo "Running scheduled task"']
          restartPolicy: OnFailure
```

### Concurrency Policies

- **Allow**: Multiple jobs can run concurrently (default)
- **Forbid**: Skip new job if previous job is still running
- **Replace**: Cancel running job and start new one

```yaml
spec:
  concurrencyPolicy: Forbid # Recommended for most cases
```

### Production Example: Regular Backups

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: database-backup
  namespace: production
spec:
  # Every day at 2 AM
  schedule: "0 2 * * *"
  timeZone: "UTC"
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 7 # Keep last 7 days
  failedJobsHistoryLimit: 3

  jobTemplate:
    spec:
      template:
        metadata:
          labels:
            app: backup
        spec:
          containers:
            - name: backup
              image: postgres:14
              command:
                - /bin/bash
                - -c
                - |
                  TIMESTAMP=$(date +%Y%m%d-%H%M%S)
                  BACKUP_FILE="/backup/db-${TIMESTAMP}.sql.gz"

                  echo "Starting backup at $(date)"
                  pg_dump -h postgres -U admin -d production | gzip > $BACKUP_FILE

                  # Upload to S3 (example)
                  aws s3 cp $BACKUP_FILE s3://backups/postgres/

                  # Clean up old local backups (keep last 3 days)
                  find /backup -name "db-*.sql.gz" -mtime +3 -delete

                  echo "Backup completed: $BACKUP_FILE"
              env:
                - name: PGPASSWORD
                  valueFrom:
                    secretKeyRef:
                      name: postgres-secret
                      key: password
                - name: AWS_ACCESS_KEY_ID
                  valueFrom:
                    secretKeyRef:
                      name: aws-credentials
                      key: access-key-id
                - name: AWS_SECRET_ACCESS_KEY
                  valueFrom:
                    secretKeyRef:
                      name: aws-credentials
                      key: secret-access-key
              volumeMounts:
                - name: backup
                  mountPath: /backup
          restartPolicy: OnFailure
          volumes:
            - name: backup
              persistentVolumeClaim:
                claimName: backup-pvc
```

### CronJob Operations

```bash
# Create CronJob
kubectl apply -f cronjob.yaml

# List CronJobs
kubectl get cronjobs
kubectl get cj  # Short form

# Describe CronJob
kubectl describe cronjob database-backup

# View Jobs created by CronJob
kubectl get jobs -l app=backup

# Manually trigger CronJob
kubectl create job --from=cronjob/database-backup manual-backup-1

# Suspend CronJob
kubectl patch cronjob database-backup -p '{"spec":{"suspend":true}}'

# Resume CronJob
kubectl patch cronjob database-backup -p '{"spec":{"suspend":false}}'

# Delete CronJob
kubectl delete cronjob database-backup
```

---

## Comparison Summary

| Feature              | Deployment     | StatefulSet   | DaemonSet           | Job              | CronJob          |
| -------------------- | -------------- | ------------- | ------------------- | ---------------- | ---------------- |
| **Purpose**          | Stateless apps | Stateful apps | Node-level tasks    | Batch processing | Scheduled tasks  |
| **Pod naming**       | Random         | Ordered       | Random              | Random           | Random           |
| **Network identity** | Dynamic        | Stable        | Dynamic             | Dynamic          | Dynamic          |
| **Storage**          | Shared/None    | Per-pod PVC   | Usually host        | Optional         | Optional         |
| **Scaling**          | Manual/Auto    | Manual        | One per node        | N/A              | N/A              |
| **Completion**       | Never          | Never         | Never               | Finite           | Finite           |
| **Use case**         | Web apps, APIs | Databases     | Monitoring, logging | Migrations       | Backups, cleanup |

---

## Best Practices

### StatefulSets

1. Always use headless service
2. Implement proper readiness probes
3. Plan for data persistence
4. Test failover scenarios
5. Use partition updates for staged rollouts

### DaemonSets

1. Set resource limits to prevent node exhaustion
2. Use tolerations for control plane nodes if needed
3. Monitor DaemonSet pod health across all nodes
4. Use hostNetwork/hostPID only when necessary
5. Implement proper logging

### Jobs

1. Set appropriate backoffLimit
2. Use ttlSecondsAfterFinished for cleanup
3. Set activeDeadlineSeconds to prevent infinite runs
4. Choose correct restartPolicy
5. Monitor Job completion and failures

### CronJobs

1. Use concurrencyPolicy: Forbid for most cases
2. Keep successfulJobsHistoryLimit reasonable
3. Set startingDeadlineSeconds for critical jobs
4. Test cron schedules before deploying
5. Monitor missed runs and failures
6. Use UTC timezone or specify explicitly

---

## Key Takeaways

1. **StatefulSets** for databases and stateful applications
2. **DaemonSets** for node-level operations
3. **Jobs** for one-time batch processing
4. **CronJobs** for scheduled recurring tasks
5. Choose the right controller for your workload
6. Understand lifecycle and behavior differences
7. Implement proper monitoring for all workload types

---

## Next Steps

👉 **[11-security-rbac.md](./11-security-rbac.md)** - Learn about Kubernetes security and access control

## Additional Resources

- [StatefulSets Documentation](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)
- [DaemonSets Documentation](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/)
- [Jobs Documentation](https://kubernetes.io/docs/concepts/workloads/controllers/job/)
- [CronJobs Documentation](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/)
