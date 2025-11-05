# ConfigMaps & Secrets

## Overview

ConfigMaps and Secrets provide ways to inject configuration data into pods, following the principle of separating configuration from application code.

## Configuration Management Philosophy

```
❌ Hard-coded Configuration:
┌──────────────────────┐
│   Application Code   │
│                      │
│  DB_HOST=db.prod.com │  ← Bad!
│  API_KEY=abc123      │  ← Bad!
│  PORT=8080           │  ← Bad!
└──────────────────────┘

✅ Externalized Configuration:
┌──────────────────────┐       ┌──────────────┐
│   Application Code   │◄──────│  ConfigMap   │
│                      │       │  - DB_HOST   │
│                      │       │  - PORT      │
└──────────────────────┘       └──────────────┘
         ▲
         │
         │                      ┌──────────────┐
         └──────────────────────│   Secret     │
                                │  - API_KEY   │
                                │  - PASSWORD  │
                                └──────────────┘
```

---

## ConfigMaps

### What is a ConfigMap?

A **ConfigMap** stores non-confidential configuration data in key-value pairs. Pods can consume ConfigMaps as environment variables, command-line arguments, or configuration files.

### Creating ConfigMaps

#### Method 1: From Literal Values

```bash
kubectl create configmap app-config \
  --from-literal=database_host=mysql.default.svc.cluster.local \
  --from-literal=database_port=3306 \
  --from-literal=environment=production
```

#### Method 2: From File

**config.properties:**

```properties
database.host=mysql.default.svc.cluster.local
database.port=3306
environment=production
max_connections=100
```

```bash
kubectl create configmap app-config --from-file=config.properties
```

#### Method 3: From Directory

```bash
# All files in directory
kubectl create configmap app-config --from-file=./configs/
```

#### Method 4: From YAML

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: default
data:
  # Simple key-value pairs
  database_host: mysql.default.svc.cluster.local
  database_port: "3306"
  environment: production

  # Multi-line data
  app.properties: |
    database.host=mysql.default.svc.cluster.local
    database.port=3306
    environment=production
    max_connections=100

  config.json: |
    {
      "database": {
        "host": "mysql.default.svc.cluster.local",
        "port": 3306
      },
      "cache": {
        "enabled": true,
        "ttl": 300
      }
    }
```

### Using ConfigMaps in Pods

#### As Environment Variables

**All keys:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: config-pod
spec:
  containers:
    - name: app
      image: myapp
      envFrom:
        - configMapRef:
            name: app-config
```

**Specific keys:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: config-pod
spec:
  containers:
    - name: app
      image: myapp
      env:
        - name: DATABASE_HOST
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: database_host
        - name: DATABASE_PORT
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: database_port
```

#### As Volume Mounts

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: config-pod
spec:
  containers:
    - name: app
      image: myapp
      volumeMounts:
        - name: config-volume
          mountPath: /etc/config

  volumes:
    - name: config-volume
      configMap:
        name: app-config
```

**Result inside container:**

```
/etc/config/
├── database_host
├── database_port
├── environment
├── app.properties
└── config.json
```

#### Mount Specific Keys

```yaml
volumes:
  - name: config-volume
    configMap:
      name: app-config
      items:
        - key: config.json
          path: application-config.json # Custom filename
        - key: app.properties
          path: app.properties
```

**Result:**

```
/etc/config/
├── application-config.json
└── app.properties
```

#### As Command Arguments

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: config-pod
spec:
  containers:
    - name: app
      image: myapp
      command: ["/bin/app"]
      args:
        - "--database-host=$(DATABASE_HOST)"
        - "--port=$(PORT)"
      env:
        - name: DATABASE_HOST
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: database_host
        - name: PORT
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: database_port
```

### ConfigMap Operations

```bash
# List ConfigMaps
kubectl get configmaps
kubectl get cm  # Short form

# Describe ConfigMap
kubectl describe configmap app-config

# View ConfigMap data
kubectl get configmap app-config -o yaml

# Edit ConfigMap
kubectl edit configmap app-config

# Delete ConfigMap
kubectl delete configmap app-config

# Create from YAML
kubectl apply -f configmap.yaml

# Update from YAML
kubectl apply -f configmap.yaml  # Same command
```

---

## Secrets

### What is a Secret?

A **Secret** stores sensitive information like passwords, tokens, and keys. Similar to ConfigMaps but specifically for confidential data.

### Secret Types

| Type                                  | Usage                                 |
| ------------------------------------- | ------------------------------------- |
| `Opaque`                              | Arbitrary user-defined data (default) |
| `kubernetes.io/service-account-token` | Service account token                 |
| `kubernetes.io/dockercfg`             | Docker config (deprecated)            |
| `kubernetes.io/dockerconfigjson`      | Docker config JSON                    |
| `kubernetes.io/basic-auth`            | Basic authentication                  |
| `kubernetes.io/ssh-auth`              | SSH authentication                    |
| `kubernetes.io/tls`                   | TLS certificate and key               |
| `bootstrap.kubernetes.io/token`       | Bootstrap tokens                      |

### Creating Secrets

#### Method 1: From Literal

```bash
kubectl create secret generic db-credentials \
  --from-literal=username=admin \
  --from-literal=password=secretpassword123
```

#### Method 2: From File

```bash
# Create files
echo -n 'admin' > username.txt
echo -n 'secretpassword123' > password.txt

# Create secret
kubectl create secret generic db-credentials \
  --from-file=username=username.txt \
  --from-file=password=password.txt

# Clean up files
rm username.txt password.txt
```

#### Method 3: From YAML (Base64 Encoded)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
data:
  # Base64 encoded values
  username: YWRtaW4= # 'admin'
  password: c2VjcmV0cGFzc3dvcmQxMjM= # 'secretpassword123'
```

**Encode values:**

```bash
echo -n 'admin' | base64
# Output: YWRtaW4=

echo -n 'secretpassword123' | base64
# Output: c2VjcmV0cGFzc3dvcmQxMjM=
```

#### Method 4: From YAML (Plain Text)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
stringData:
  # Plain text (automatically encoded)
  username: admin
  password: secretpassword123
```

### Specialized Secret Types

#### Docker Registry Secret

```bash
kubectl create secret docker-registry my-registry \
  --docker-server=registry.example.com \
  --docker-username=myuser \
  --docker-password=mypassword \
  --docker-email=user@example.com
```

**Use in Pod:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: private-image-pod
spec:
  containers:
    - name: app
      image: registry.example.com/myapp:latest
  imagePullSecrets:
    - name: my-registry
```

#### TLS Secret

```bash
kubectl create secret tls tls-secret \
  --cert=path/to/tls.crt \
  --key=path/to/tls.key
```

**YAML:**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: tls-secret
type: kubernetes.io/tls
data:
  tls.crt: <base64-encoded-cert>
  tls.key: <base64-encoded-key>
```

**Use in Ingress:**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
spec:
  tls:
    - hosts:
        - example.com
      secretName: tls-secret
  rules:
    - host: example.com
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

#### Basic Auth Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: basic-auth
type: kubernetes.io/basic-auth
stringData:
  username: admin
  password: secretpassword
```

#### SSH Auth Secret

```bash
kubectl create secret generic ssh-key \
  --from-file=ssh-privatekey=~/.ssh/id_rsa \
  --type=kubernetes.io/ssh-auth
```

### Using Secrets in Pods

#### As Environment Variables

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-pod
spec:
  containers:
    - name: app
      image: myapp
      env:
        - name: DB_USERNAME
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: username
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password
```

**Load all keys:**

```yaml
spec:
  containers:
    - name: app
      image: myapp
      envFrom:
        - secretRef:
            name: db-credentials
```

#### As Volume Mounts

```yaml
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
        defaultMode: 0400 # Read-only for owner
```

**Result inside container:**

```
/etc/secrets/
├── username
└── password
```

### Secret Operations

```bash
# List secrets
kubectl get secrets

# Describe secret (values hidden)
kubectl describe secret db-credentials

# View secret (base64 encoded)
kubectl get secret db-credentials -o yaml

# Decode secret value
kubectl get secret db-credentials -o jsonpath='{.data.password}' | base64 --decode

# Edit secret
kubectl edit secret db-credentials

# Delete secret
kubectl delete secret db-credentials
```

---

## ConfigMaps vs Secrets

| Feature        | ConfigMap                | Secret                     |
| -------------- | ------------------------ | -------------------------- |
| **Purpose**    | Non-sensitive config     | Sensitive data             |
| **Storage**    | Plain text               | Base64 encoded             |
| **At Rest**    | Not encrypted by default | Not encrypted by default\* |
| **In Transit** | Not encrypted            | Not encrypted              |
| **Size Limit** | 1MB                      | 1MB                        |
| **Use For**    | Configuration, settings  | Passwords, keys, tokens    |

**Note:** Both need additional configuration for encryption at rest.

---

## Best Practices

### ConfigMap Best Practices

1. **Separate config from code**

   ```yaml
   # Good: External config
   env:
   - name: DATABASE_HOST
     valueFrom:
       configMapKeyRef:
         name: app-config
         key: database_host

   # Bad: Hard-coded
   env:
   - name: DATABASE_HOST
     value: "mysql.prod.com"
   ```

2. **Use descriptive names**

   ```yaml
   # Good
   configmap-app-database-dev
   configmap-app-database-prod

   # Bad
   config1
   config2
   ```

3. **Version your ConfigMaps**

   ```yaml
   metadata:
     name: app-config-v2
   ```

4. **Document your configs**
   ```yaml
   metadata:
     annotations:
       description: "Application configuration for production environment"
       version: "2.0"
       updated: "2024-01-15"
   ```

### Secret Best Practices

1. **Never commit secrets to version control**

   ```bash
   # .gitignore
   secrets/
   *.secret
   ```

2. **Use RBAC to restrict access**

   ```yaml
   apiVersion: rbac.authorization.k8s.io/v1
   kind: Role
   metadata:
     name: secret-reader
   rules:
     - apiGroups: [""]
       resources: ["secrets"]
       resourceNames: ["db-credentials"]
       verbs: ["get"]
   ```

3. **Enable encryption at rest**

   ```yaml
   # EncryptionConfiguration
   apiVersion: apiserver.config.k8s.io/v1
   kind: EncryptionConfiguration
   resources:
     - resources:
         - secrets
       providers:
         - aescbc:
             keys:
               - name: key1
                 secret: <base64-encoded-key>
         - identity: {}
   ```

4. **Use external secret management**

   - HashiCorp Vault
   - AWS Secrets Manager
   - Azure Key Vault
   - Google Secret Manager

5. **Rotate secrets regularly**

   ```bash
   # Update secret
   kubectl create secret generic db-credentials \
     --from-literal=password=newpassword123 \
     --dry-run=client -o yaml | kubectl apply -f -

   # Restart pods to pick up new secret
   kubectl rollout restart deployment myapp
   ```

6. **Use least privilege**

   ```yaml
   # Don't expose all secrets
   envFrom:
     - secretRef:
         name: all-secrets # ❌ Avoid

   # Expose only what's needed
   env:
     - name: DB_PASSWORD
       valueFrom:
         secretKeyRef:
           name: db-credentials
           key: password # ✅ Better
   ```

---

## Advanced Patterns

### Immutable ConfigMaps & Secrets

Prevent accidental updates (Kubernetes 1.21+).

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: immutable-config
immutable: true
data:
  key: value
```

**Benefits:**

- Protects from accidental updates
- Improves performance (no watches needed)
- Must create new ConfigMap to update

### Multi-Environment Configuration

```yaml
# dev-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: dev
data:
  environment: development
  database_host: mysql.dev.svc.cluster.local
  log_level: debug
---
# prod-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: prod
data:
  environment: production
  database_host: mysql.prod.svc.cluster.local
  log_level: info
```

### Combining ConfigMaps and Secrets

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
    - name: app
      image: myapp
      env:
        # From ConfigMap
        - name: DATABASE_HOST
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: database_host
        # From Secret
        - name: DATABASE_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password
```

### Configuration Hierarchy

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app
    image: myapp
    # 1. Direct values (highest priority)
    env:
    - name: LOG_LEVEL
      value: "debug"
    # 2. ConfigMap values
    envFrom:
    - configMapRef:
        name: app-config
    # 3. Secret values
    envFrom:
    - secretRef:
        name: app-secrets
```

---

## Practical Examples

### Example 1: Web Application with Database

```yaml
# ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: webapp-config
data:
  nginx.conf: |
    server {
      listen 80;
      server_name example.com;
      
      location / {
        proxy_pass http://app:8080;
      }
    }
  app.properties: |
    database.host=mysql
    database.port=3306
    database.name=webapp
    cache.enabled=true
    cache.ttl=300
---
# Secret
apiVersion: v1
kind: Secret
metadata:
  name: webapp-secrets
type: Opaque
stringData:
  database-password: supersecretpassword
  api-key: abc123xyz789
---
# Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
        - name: app
          image: mywebapp:1.0
          env:
            # From ConfigMap
            - name: DATABASE_HOST
              valueFrom:
                configMapKeyRef:
                  name: webapp-config
                  key: database.host
            # From Secret
            - name: DATABASE_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: webapp-secrets
                  key: database-password
            - name: API_KEY
              valueFrom:
                secretKeyRef:
                  name: webapp-secrets
                  key: api-key
          volumeMounts:
            - name: config
              mountPath: /etc/config

      volumes:
        - name: config
          configMap:
            name: webapp-config
            items:
              - key: nginx.conf
                path: nginx.conf
              - key: app.properties
                path: application.properties
```

### Example 2: External Secret Operator

Using External Secrets Operator to sync from AWS Secrets Manager:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-secretsmanager
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets
---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: database-credentials
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secretsmanager
    kind: SecretStore
  target:
    name: db-credentials
    creationPolicy: Owner
  data:
    - secretKey: username
      remoteRef:
        key: prod/database
        property: username
    - secretKey: password
      remoteRef:
        key: prod/database
        property: password
```

---

## Troubleshooting

### Common Issues

**1. ConfigMap/Secret not found**

```bash
# Check if exists
kubectl get configmap app-config
kubectl get secret db-credentials

# Check namespace
kubectl get configmap -n <namespace>
```

**2. Environment variables not updating**

```bash
# ConfigMaps/Secrets are not hot-reloaded by default
# Restart pods to pick up changes
kubectl rollout restart deployment myapp
```

**3. Permission denied accessing secrets**

```bash
# Check RBAC permissions
kubectl auth can-i get secrets --as=system:serviceaccount:default:myapp

# Check service account
kubectl describe sa myapp
```

**4. Decode secret for debugging**

```bash
# Get base64 value
kubectl get secret db-credentials -o jsonpath='{.data.password}'

# Decode
kubectl get secret db-credentials -o jsonpath='{.data.password}' | base64 --decode
```

---

## Key Takeaways

1. **ConfigMaps** for non-sensitive configuration
2. **Secrets** for sensitive data (passwords, keys)
3. Both are **limited to 1MB**
4. Use **volume mounts** for files, **env vars** for simple values
5. Secrets are **base64 encoded**, not encrypted by default
6. **Enable encryption at rest** for production
7. Consider **external secret management** for enhanced security
8. **Don't commit secrets** to version control
9. **Rotate secrets** regularly
10. Use **RBAC** to restrict secret access

---

## Next Steps

👉 **[10-advanced-workloads.md](./10-advanced-workloads.md)** - Learn about StatefulSets, DaemonSets, Jobs, and CronJobs

## Additional Resources

- [ConfigMaps Documentation](https://kubernetes.io/docs/concepts/configuration/configmap/)
- [Secrets Documentation](https://kubernetes.io/docs/concepts/configuration/secret/)
- [Encrypting Secret Data at Rest](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/)
- [External Secrets Operator](https://external-secrets.io/)
