# Security & RBAC

## Overview

Kubernetes security involves multiple layers: authentication, authorization (RBAC), network policies, pod security, and secrets management. This guide covers implementing a comprehensive security posture.

## Security Layers

```
┌─────────────────────────────────────────────┐
│         1. API Server Security              │
│     (Authentication & Authorization)        │
├─────────────────────────────────────────────┤
│         2. RBAC                             │
│     (Role-Based Access Control)             │
├─────────────────────────────────────────────┤
│         3. Network Policies                 │
│     (Pod-to-Pod Communication)              │
├─────────────────────────────────────────────┤
│         4. Pod Security                     │
│     (Security Contexts & Standards)         │
├─────────────────────────────────────────────┤
│         5. Secrets Management               │
│     (Encrypted Data at Rest)                │
├─────────────────────────────────────────────┤
│         6. Image Security                   │
│     (Image Scanning & Policies)             │
└─────────────────────────────────────────────┘
```

---

## Authentication

### Authentication Methods

1. **Client Certificates** (X509)
2. **Bearer Tokens** (Service Account tokens)
3. **OIDC Tokens** (OpenID Connect)
4. **Webhook Token Authentication**
5. **Authentication Proxy**

### Service Accounts

**Default Service Account:**

```bash
# Every namespace has a default SA
kubectl get serviceaccount
kubectl get sa  # Short form

# View default SA
kubectl get sa default -o yaml
```

**Create Service Account:**

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-service-account
  namespace: default
```

**Use in Pod:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  serviceAccountName: my-service-account
  containers:
    - name: app
      image: myapp
```

**Disable Auto-mount:**

```yaml
spec:
  serviceAccountName: my-service-account
  automountServiceAccountToken: false
```

---

## RBAC (Role-Based Access Control)

### RBAC Components

```
User/ServiceAccount
        │
        ▼
    RoleBinding ──────► Role (namespace-scoped)
        or                  or
ClusterRoleBinding ──► ClusterRole (cluster-wide)
        │
        ▼
   Permissions on Resources
```

### Roles & ClusterRoles

**Role (Namespace-scoped):**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: default
rules:
  - apiGroups: [""] # "" indicates core API group
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["pods/log"]
    verbs: ["get"]
```

**ClusterRole (Cluster-wide):**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: secret-reader
rules:
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get", "list"]
```

**Verbs (Permissions):**

- `get`: Read single resource
- `list`: List resources
- `watch`: Watch for changes
- `create`: Create resource
- `update`: Update resource
- `patch`: Patch resource
- `delete`: Delete resource
- `deletecollection`: Delete collection

### RoleBindings & ClusterRoleBindings

**RoleBinding (Binds Role to subjects in namespace):**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:
  - kind: User
    name: jane
    apiGroup: rbac.authorization.k8s.io
  - kind: ServiceAccount
    name: my-service-account
    namespace: default
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

**ClusterRoleBinding (Cluster-wide binding):**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: read-secrets-global
subjects:
  - kind: Group
    name: system:authenticated
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```

### Common RBAC Patterns

#### Pattern 1: Namespace Admin

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: namespace-admin
  namespace: development
rules:
  - apiGroups: ["", "apps", "batch"]
    resources: ["*"]
    verbs: ["*"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-admin-binding
  namespace: development
subjects:
  - kind: User
    name: dev-lead
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: namespace-admin
  apiGroup: rbac.authorization.k8s.io
```

#### Pattern 2: Read-Only Access

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: read-only
rules:
  - apiGroups: ["", "apps", "batch", "networking.k8s.io"]
    resources: ["*"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: read-only-binding
subjects:
  - kind: Group
    name: viewers
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: read-only
  apiGroup: rbac.authorization.k8s.io
```

#### Pattern 3: CI/CD Service Account

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ci-deployer
  namespace: production
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: deployer
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
  name: ci-deployer-binding
  namespace: production
subjects:
  - kind: ServiceAccount
    name: ci-deployer
    namespace: production
roleRef:
  kind: Role
  name: deployer
  apiGroup: rbac.authorization.k8s.io
```

### RBAC Commands

```bash
# Create Role
kubectl create role pod-reader --verb=get,list --resource=pods

# Create ClusterRole
kubectl create clusterrole secret-reader --verb=get,list --resource=secrets

# Create RoleBinding
kubectl create rolebinding read-pods --role=pod-reader --user=jane

# Create ClusterRoleBinding
kubectl create clusterrolebinding read-secrets --clusterrole=secret-reader --user=jane

# Check if user can perform action
kubectl auth can-i get pods --as=jane
kubectl auth can-i delete deployments --as=system:serviceaccount:default:my-sa

# Check all permissions for user
kubectl auth can-i --list --as=jane

# View Role
kubectl get role pod-reader -o yaml

# View RoleBinding
kubectl get rolebinding read-pods -o yaml

# Delete RBAC resources
kubectl delete role pod-reader
kubectl delete rolebinding read-pods
```

---

## Pod Security

### Security Contexts

**Pod-level Security Context:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
    runAsNonRoot: true
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: app
      image: myapp
```

**Container-level Security Context:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  containers:
    - name: app
      image: myapp
      securityContext:
        allowPrivilegeEscalation: false
        runAsNonRoot: true
        runAsUser: 1000
        capabilities:
          drop:
            - ALL
          add:
            - NET_BIND_SERVICE
        readOnlyRootFilesystem: true
```

### Security Context Options

```yaml
securityContext:
  # Run as non-root user
  runAsNonRoot: true
  runAsUser: 1000
  runAsGroup: 3000

  # Filesystem group
  fsGroup: 2000

  # Prevent privilege escalation
  allowPrivilegeEscalation: false

  # Run as privileged (avoid!)
  privileged: false

  # Read-only root filesystem
  readOnlyRootFilesystem: true

  # Linux capabilities
  capabilities:
    drop:
      - ALL
    add:
      - NET_BIND_SERVICE

  # SELinux options
  seLinuxOptions:
    level: "s0:c123,c456"

  # Seccomp profile
  seccompProfile:
    type: RuntimeDefault
```

### Pod Security Standards

Three levels of Pod Security:

1. **Privileged**: Unrestricted (default)
2. **Baseline**: Minimal restrictions
3. **Restricted**: Heavily restricted

**Apply at namespace level:**

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: secure-ns
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

---

## Network Policies

### Default Deny All

```yaml
# Deny all ingress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
---
# Deny all egress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-egress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Egress
```

### Allow Specific Traffic

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-network-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
    - Ingress
    - Egress

  ingress:
    # Allow from frontend
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 8080

  egress:
    # Allow to database
    - to:
        - podSelector:
            matchLabels:
              app: database
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

---

## Secrets Encryption

### Enable Encryption at Rest

**EncryptionConfiguration:**

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: <base64-encoded-32-byte-key>
      - identity: {}
```

**Generate encryption key:**

```bash
head -c 32 /dev/urandom | base64
```

**Apply to API server:**

```bash
# Add to kube-apiserver flags
--encryption-provider-config=/etc/kubernetes/enc/encryption-config.yaml
```

---

## Image Security

### Image Pull Secrets

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: regcred
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: <base64-encoded-config>
---
apiVersion: v1
kind: Pod
metadata:
  name: private-image-pod
spec:
  containers:
    - name: app
      image: private-registry.com/myapp:1.0
  imagePullSecrets:
    - name: regcred
```

### Image Policy Webhook

```yaml
# Admission configuration
apiVersion: apiserver.config.k8s.io/v1
kind: AdmissionConfiguration
plugins:
  - name: ImagePolicyWebhook
    configuration:
      imagePolicy:
        kubeConfigFile: /path/to/kubeconfig
        allowTTL: 50
        denyTTL: 50
        retryBackoff: 500
        defaultAllow: false
```

---

## Admission Controllers

### Common Admission Controllers

- **PodSecurityPolicy**: Enforce security standards (deprecated, use Pod Security Standards)
- **ResourceQuota**: Enforce resource quotas
- **LimitRanger**: Enforce resource limits
- **NamespaceLifecycle**: Prevent deletion of system namespaces
- **ServiceAccount**: Automate service account assignment
- **ValidatingAdmissionWebhook**: Custom validation
- **MutatingAdmissionWebhook**: Custom mutation

### Example: Validating Webhook

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: pod-policy
webhooks:
  - name: pod-policy.example.com
    clientConfig:
      service:
        name: pod-policy-webhook
        namespace: default
        path: "/validate"
      caBundle: <base64-encoded-ca-cert>
    rules:
      - operations: ["CREATE"]
        apiGroups: [""]
        apiVersions: ["v1"]
        resources: ["pods"]
    admissionReviewVersions: ["v1"]
    sideEffects: None
```

---

## Security Best Practices

### 1. Least Privilege

```yaml
# ❌ Bad: Too permissive
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]

# ✅ Good: Specific permissions
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "update"]
```

### 2. Non-Root Containers

```yaml
# ✅ Good
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  allowPrivilegeEscalation: false
```

### 3. Read-Only Root Filesystem

```yaml
securityContext:
  readOnlyRootFilesystem: true
volumeMounts:
  - name: tmp
    mountPath: /tmp
volumes:
  - name: tmp
    emptyDir: {}
```

### 4. Drop Capabilities

```yaml
securityContext:
  capabilities:
    drop:
      - ALL
```

### 5. Network Policies

```yaml
# Start with deny-all
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
```

### 6. Resource Limits

```yaml
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi
```

### 7. Secrets Management

```yaml
# Use external secrets
# - HashiCorp Vault
# - AWS Secrets Manager
# - Azure Key Vault
# - Google Secret Manager
```

### 8. Regular Updates

```bash
# Keep Kubernetes updated
# Scan images regularly
# Monitor CVEs
```

---

## Security Audit

### Audit Log Example

```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  # Log all requests at RequestResponse level
  - level: RequestResponse
    resources:
      - group: ""
        resources: ["secrets"]

  # Log pod changes at Request level
  - level: Request
    resources:
      - group: ""
        resources: ["pods"]
    verbs: ["create", "update", "delete"]
```

---

## Security Tools

### 1. Kube-bench

Checks Kubernetes deployment against CIS benchmark.

```bash
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job.yaml
kubectl logs -f job/kube-bench
```

### 2. Kubesec

Static analysis of Kubernetes manifests.

```bash
# Scan manifest
kubesec scan deployment.yaml

# HTTP API
curl -sSX POST --data-binary @"deployment.yaml" https://v2.kubesec.io/scan
```

### 3. Falco

Runtime security monitoring.

```bash
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm install falco falcosecurity/falco
```

### 4. OPA (Open Policy Agent)

Policy enforcement.

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: opa-validating-webhook
webhooks:
  - name: validating-webhook.openpolicyagent.org
    # ... OPA configuration
```

---

## Compliance & Certifications

### CIS Benchmark

```bash
# Run kube-bench
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job.yaml
```

### Pod Security Standards

```yaml
# Apply to namespace
kubectl label namespace production \
pod-security.kubernetes.io/enforce=restricted \
pod-security.kubernetes.io/audit=restricted \
pod-security.kubernetes.io/warn=restricted
```

---

## Key Takeaways

1. **Implement RBAC** with least privilege principle
2. **Use Security Contexts** to harden pods
3. **Enable Network Policies** to control traffic
4. **Encrypt Secrets** at rest
5. **Scan images** for vulnerabilities
6. **Use Pod Security Standards**
7. **Audit and monitor** cluster activity
8. **Regular security assessments**
9. **Keep Kubernetes updated**
10. **Follow security best practices**

---

## Next Steps

👉 **[12-monitoring-logging.md](./12-monitoring-logging.md)** - Learn about monitoring and observability

## Additional Resources

- [Kubernetes Security Best Practices](https://kubernetes.io/docs/concepts/security/)
- [RBAC Documentation](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
- [Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [CIS Benchmark](https://www.cisecurity.org/benchmark/kubernetes)
