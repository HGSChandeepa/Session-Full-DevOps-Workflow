# Storage & Volumes

## Overview

Kubernetes provides multiple storage options for persisting data beyond the lifecycle of individual pods. Understanding volumes and persistent storage is crucial for stateful applications.

## The Storage Problem

```
Pod Lifecycle:
┌──────────┐      ┌──────────┐      ┌──────────┐
│  Pod v1  │ ───> │  Pod v2  │ ───> │  Pod v3  │
│  Data ❌ │      │  Data ❌ │      │  Data ❌ │
└──────────┘      └──────────┘      └──────────┘
   Lost!            Lost!            Lost!

With Volumes:
┌──────────┐      ┌──────────┐      ┌──────────┐
│  Pod v1  │      │  Pod v2  │      │  Pod v3  │
└──────────┘      └──────────┘      └──────────┘
     │                 │                 │
     └─────────────────┴─────────────────┘
                       │
                ┌──────▼──────┐
                │   Volume    │
                │   Data ✅   │
                └─────────────┘
                 Persistent!
```

---

## Volume Types

### 1. EmptyDir

Temporary volume that exists as long as the pod exists. Deleted when pod is removed.

**Use Cases:**

- Temporary scratch space
- Cache
- Sharing data between containers in same pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: emptydir-pod
spec:
  containers:
    - name: writer
      image: busybox
      command: ["sh", "-c", 'echo "Hello" > /data/hello.txt; sleep 3600']
      volumeMounts:
        - name: cache
          mountPath: /data

    - name: reader
      image: busybox
      command: ["sh", "-c", "cat /data/hello.txt; sleep 3600"]
      volumeMounts:
        - name: cache
          mountPath: /data

  volumes:
    - name: cache
      emptyDir: {}
```

**EmptyDir with Memory:**

```yaml
volumes:
  - name: cache
    emptyDir:
      medium: Memory # Use tmpfs (RAM)
      sizeLimit: 128Mi
```

---

### 2. HostPath

Mounts file or directory from host node filesystem into pod.

**Use Cases:**

- Access node logs
- Docker socket access
- Node monitoring

**⚠️ Warning:** Security risk, breaks pod portability, avoid in production.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-pod
spec:
  containers:
    - name: app
      image: nginx
      volumeMounts:
        - name: host-logs
          mountPath: /var/log/host

  volumes:
    - name: host-logs
      hostPath:
        path: /var/log
        type: Directory
```

**HostPath Types:**

- `Directory`: Must exist
- `DirectoryOrCreate`: Create if doesn't exist
- `File`: Must exist
- `FileOrCreate`: Create if doesn't exist
- `Socket`: Unix socket must exist
- `CharDevice`, `BlockDevice`: Device files

---

### 3. ConfigMap

Mount ConfigMap as volume for configuration files.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  config.json: |
    {
      "database": "postgres",
      "port": 5432
    }
  app.properties: |
    env=production
    debug=false
---
apiVersion: v1
kind: Pod
metadata:
  name: configmap-pod
spec:
  containers:
    - name: app
      image: myapp
      volumeMounts:
        - name: config
          mountPath: /etc/config

  volumes:
    - name: config
      configMap:
        name: app-config
```

**Mount Specific Keys:**

```yaml
volumes:
  - name: config
    configMap:
      name: app-config
      items:
        - key: config.json
          path: app-config.json # Rename file
```

---

### 4. Secret

Mount Secret as volume for sensitive data.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
data:
  username: YWRtaW4= # base64 encoded
  password: cGFzc3dvcmQ=
---
apiVersion: v1
kind: Pod
metadata:
  name: secret-pod
spec:
  containers:
    - name: app
      image: myapp
      volumeMounts:
        - name: secrets
          mountPath: /etc/secrets
          readOnly: true

  volumes:
    - name: secrets
      secret:
        secretName: db-credentials
```

**Default Permission:** 0644 (readable by all)

**Custom Permission:**

```yaml
volumes:
  - name: secrets
    secret:
      secretName: db-credentials
      defaultMode: 0400 # Read-only for owner
```

---

### 5. Persistent Volumes (PV) & Persistent Volume Claims (PVC)

Decouples storage provisioning from consumption.

#### Architecture

```
┌──────────────────────────────────────────────┐
│         Storage Admin                        │
│                                              │
│   Creates PersistentVolumes (PV)            │
│   ┌────────┐  ┌────────┐  ┌────────┐       │
│   │ PV-1   │  │ PV-2   │  │ PV-3   │       │
│   │ 10Gi   │  │ 50Gi   │  │ 100Gi  │       │
│   └────────┘  └────────┘  └────────┘       │
└──────────────────────────────────────────────┘
                    │
                Binding
                    │
┌──────────────────────────────────────────────┐
│         Developer                            │
│                                              │
│   Creates PersistentVolumeClaims (PVC)      │
│   ┌────────┐                                │
│   │ PVC-1  │───────┐                        │
│   │ 20Gi   │       │                        │
│   └────────┘       │                        │
│                    ▼                         │
│               ┌────────┐                     │
│               │  Pod   │                     │
│               └────────┘                     │
└──────────────────────────────────────────────┘
```

#### PersistentVolume (PV)

Cluster resource provisioned by admin.

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-example
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath:
    path: /mnt/data
```

**Access Modes:**

- `ReadWriteOnce` (RWO): One node can mount read-write
- `ReadOnlyMany` (ROX): Many nodes can mount read-only
- `ReadWriteMany` (RWX): Many nodes can mount read-write

**Reclaim Policies:**

- `Retain`: Manual reclamation (data preserved)
- `Delete`: Delete volume and data
- `Recycle`: Scrub data, available for new claim (deprecated)

#### PersistentVolumeClaim (PVC)

Request for storage by user.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-example
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: manual
```

#### Using PVC in Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pvc-pod
spec:
  containers:
    - name: app
      image: nginx
      volumeMounts:
        - name: storage
          mountPath: /usr/share/nginx/html

  volumes:
    - name: storage
      persistentVolumeClaim:
        claimName: pvc-example
```

#### PV Lifecycle

```
┌──────────────┐
│ Provisioning │ ← Admin creates PV or dynamic provisioning
└──────┬───────┘
       │
       ▼
┌──────────────┐
│   Binding    │ ← PVC binds to matching PV
└──────┬───────┘
       │
       ▼
┌──────────────┐
│    Using     │ ← Pod uses PVC
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  Releasing   │ ← PVC deleted, PV released
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  Reclaiming  │ ← Based on reclaim policy
└──────────────┘
```

---

## StorageClass

Enables dynamic provisioning of PersistentVolumes.

### Default StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-storage
provisioner: kubernetes.io/gce-pd # Cloud provider
parameters:
  type: pd-ssd
  replication-type: regional-pd
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```

**Provisioners:**

- `kubernetes.io/aws-ebs`: AWS EBS
- `kubernetes.io/azure-disk`: Azure Disk
- `kubernetes.io/gce-pd`: Google Persistent Disk
- `kubernetes.io/cinder`: OpenStack Cinder
- `kubernetes.io/vsphere-volume`: vSphere
- `kubernetes.io/no-provisioner`: Manual provisioning

### Using StorageClass

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: fast-storage # References StorageClass
  resources:
    requests:
      storage: 10Gi
```

**No StorageClass (Manual):**

```yaml
spec:
  storageClassName: "" # Disable dynamic provisioning
```

### Volume Binding Modes

**Immediate:**

- Volume provisioned immediately after PVC creation
- Pod can be scheduled on any node

**WaitForFirstConsumer:**

- Volume provisioned when pod using PVC is scheduled
- Ensures volume is in same zone as pod

```yaml
volumeBindingMode: WaitForFirstConsumer # Recommended
```

---

## StatefulSets with Storage

StatefulSets automatically create PVCs for each replica.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: database
spec:
  serviceName: database
  replicas: 3
  selector:
    matchLabels:
      app: db
  template:
    metadata:
      labels:
        app: db
    spec:
      containers:
        - name: postgres
          image: postgres:14
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data

  # Volume claim template
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes:
          - ReadWriteOnce
        storageClassName: fast-storage
        resources:
          requests:
            storage: 10Gi
```

**Creates:**

- `data-database-0` (PVC for pod 0)
- `data-database-1` (PVC for pod 1)
- `data-database-2` (PVC for pod 2)

---

## Cloud Provider Examples

### AWS EBS

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: aws-ebs
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp3
  iopsPerGB: "10"
  fsType: ext4
  encrypted: "true"
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```

### Google Persistent Disk

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gcp-pd
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-ssd
  replication-type: regional-pd
allowVolumeExpansion: true
```

### Azure Disk

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: azure-disk
provisioner: kubernetes.io/azure-disk
parameters:
  storageaccounttype: Premium_LRS
  kind: Managed
```

---

## NFS Volumes

For ReadWriteMany access.

### NFS Server Setup (Example)

```yaml
# NFS Server Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-server
spec:
  replicas: 1
  selector:
    matchLabels:
      role: nfs-server
  template:
    metadata:
      labels:
        role: nfs-server
    spec:
      containers:
        - name: nfs-server
          image: k8s.gcr.io/volume-nfs:0.8
          ports:
            - name: nfs
              containerPort: 2049
            - name: mountd
              containerPort: 20048
            - name: rpcbind
              containerPort: 111
          securityContext:
            privileged: true
---
# NFS Service
apiVersion: v1
kind: Service
metadata:
  name: nfs-server
spec:
  ports:
    - name: nfs
      port: 2049
    - name: mountd
      port: 20048
    - name: rpcbind
      port: 111
  selector:
    role: nfs-server
```

### NFS PersistentVolume

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteMany
  nfs:
    server: nfs-server.default.svc.cluster.local
    path: "/exports"
```

---

## Volume Snapshots

Create point-in-time copies of volumes.

### VolumeSnapshotClass

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: csi-snapclass
driver: pd.csi.storage.gke.io
deletionPolicy: Delete
```

### VolumeSnapshot

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: pvc-snapshot
spec:
  volumeSnapshotClassName: csi-snapclass
  source:
    persistentVolumeClaimName: pvc-example
```

### Restore from Snapshot

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: restored-pvc
spec:
  dataSource:
    name: pvc-snapshot
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

---

## Practical Examples

### Example 1: WordPress with MySQL

```yaml
# MySQL PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
---
# MySQL Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
        - name: mysql
          image: mysql:8.0
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: password
          volumeMounts:
            - name: mysql-storage
              mountPath: /var/lib/mysql
      volumes:
        - name: mysql-storage
          persistentVolumeClaim:
            claimName: mysql-pvc
---
# WordPress PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: wordpress-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
---
# WordPress Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
spec:
  replicas: 1
  selector:
    matchLabels:
      app: wordpress
  template:
    metadata:
      labels:
        app: wordpress
    spec:
      containers:
        - name: wordpress
          image: wordpress:latest
          env:
            - name: WORDPRESS_DB_HOST
              value: mysql
            - name: WORDPRESS_DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: password
          volumeMounts:
            - name: wordpress-storage
              mountPath: /var/www/html
      volumes:
        - name: wordpress-storage
          persistentVolumeClaim:
            claimName: wordpress-pvc
```

### Example 2: Shared Storage Across Pods

```yaml
# PVC with ReadWriteMany
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 5Gi
  storageClassName: nfs
---
# Deployment using shared storage
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-servers
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
          volumeMounts:
            - name: shared-data
              mountPath: /usr/share/nginx/html
      volumes:
        - name: shared-data
          persistentVolumeClaim:
            claimName: shared-pvc
```

---

## Storage Operations

### PersistentVolume Commands

```bash
# List PVs
kubectl get pv

# Describe PV
kubectl describe pv <pv-name>

# Delete PV (if not bound)
kubectl delete pv <pv-name>

# Patch PV reclaim policy
kubectl patch pv <pv-name> -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'
```

### PersistentVolumeClaim Commands

```bash
# List PVCs
kubectl get pvc

# Describe PVC
kubectl describe pvc <pvc-name>

# Delete PVC
kubectl delete pvc <pvc-name>

# Expand PVC (if allowVolumeExpansion: true)
kubectl patch pvc <pvc-name> -p '{"spec":{"resources":{"requests":{"storage":"20Gi"}}}}'
```

### StorageClass Commands

```bash
# List storage classes
kubectl get storageclass
kubectl get sc  # Short form

# Describe storage class
kubectl describe storageclass <sc-name>

# Set default storage class
kubectl patch storageclass <sc-name> \
  -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

---

## Troubleshooting

### Common Issues

**1. PVC Pending**

```bash
# Check events
kubectl describe pvc <pvc-name>

# Common causes:
# - No matching PV available
# - StorageClass doesn't exist
# - Insufficient storage capacity
# - Access mode mismatch
```

**2. Volume Mount Failures**

```bash
# Check pod events
kubectl describe pod <pod-name>

# Check PVC status
kubectl get pvc

# Common causes:
# - PVC not bound
# - Wrong access mode
# - Volume already mounted elsewhere (RWO)
```

**3. Permission Issues**

```bash
# Check from within pod
kubectl exec -it <pod-name> -- ls -la /mount/path

# Fix with initContainer
spec:
  initContainers:
  - name: fix-permissions
    image: busybox
    command: ['sh', '-c', 'chown -R 1000:1000 /data']
    volumeMounts:
    - name: data
      mountPath: /data
```

---

## Best Practices

1. **Use StorageClasses** for dynamic provisioning
2. **Set resource requests** appropriately
3. **Choose correct access mode** (RWO vs RWX)
4. **Enable volume expansion** when possible
5. **Use WaitForFirstConsumer** for topology-aware provisioning
6. **Backup important data** regularly
7. **Monitor storage usage** and costs
8. **Use appropriate reclaim policy** (Retain for production)
9. **Encrypt sensitive data** at rest
10. **Test restore procedures** regularly

---

## Key Takeaways

1. **Volumes** persist data beyond pod lifecycle
2. **PV/PVC** decouples storage from pods
3. **StorageClass** enables dynamic provisioning
4. **Access modes** determine how volumes can be shared
5. **StatefulSets** automate storage for stateful apps
6. Choose **appropriate volume type** for use case
7. Always **test backup/restore** procedures

---

## Next Steps

👉 **[09-configmaps-secrets.md](./09-configmaps-secrets.md)** - Manage configuration and sensitive data

## Additional Resources

- [Volumes Documentation](https://kubernetes.io/docs/concepts/storage/volumes/)
- [Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [Storage Classes](https://kubernetes.io/docs/concepts/storage/storage-classes/)
- [StatefulSets](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)
