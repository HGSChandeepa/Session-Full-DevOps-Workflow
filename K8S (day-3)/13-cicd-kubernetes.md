# CI/CD with Kubernetes

## Overview

Implementing CI/CD for Kubernetes involves automating the build, test, and deployment pipeline. This guide covers various tools and strategies for continuous delivery to Kubernetes clusters.

## CI/CD Pipeline Architecture

```
┌─────────────────────────────────────────────────────┐
│                    CI/CD PIPELINE                    │
├─────────────────────────────────────────────────────┤
│                                                      │
│  1. Source Code                                     │
│     ├── Git Push                                    │
│     └── Webhook Trigger                             │
│          │                                           │
│          ▼                                           │
│  2. Build                                           │
│     ├── Run Tests                                   │
│     ├── Build Docker Image                          │
│     └── Push to Registry                            │
│          │                                           │
│          ▼                                           │
│  3. Deploy                                          │
│     ├── Update K8s Manifests                        │
│     ├── Apply to Cluster                            │
│     └── Verify Deployment                           │
│          │                                           │
│          ▼                                           │
│  4. Monitor                                         │
│     ├── Check Health                                │
│     ├── Monitor Metrics                             │
│     └── Alerts if Issues                            │
│                                                      │
└─────────────────────────────────────────────────────┘
```

---

## GitOps Principles

### What is GitOps?

**GitOps** is a way of implementing Continuous Deployment where Git is the single source of truth for declarative infrastructure and applications.

**Core Principles:**

1. **Declarative**: Everything in Git (YAML manifests)
2. **Versioned**: Git history tracks all changes
3. **Automated**: Changes auto-applied to cluster
4. **Reconciled**: Cluster state matches Git state

**GitOps Workflow:**

```
Developer
    │
    │ 1. Push code
    ▼
┌──────────┐
│   Git    │
└────┬─────┘
     │ 2. Webhook/Poll
     ▼
┌──────────┐
│ GitOps   │
│ Operator │
└────┬─────┘
     │ 3. Apply
     ▼
┌──────────┐
│   K8s    │
│ Cluster  │
└──────────┘
```

---

## ArgoCD (GitOps Tool)

### Installation

```bash
# Create namespace
kubectl create namespace argocd

# Install ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for pods
kubectl wait --for=condition=Ready pods --all -n argocd --timeout=300s

# Get initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d

# Port forward to access UI
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Access: https://localhost:8080
# Username: admin
# Password: <from above>
```

### ArgoCD Application

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
spec:
  # Project
  project: default

  # Source (Git repository)
  source:
    repoURL: https://github.com/myorg/myapp
    targetRevision: main
    path: k8s/overlays/production

  # Destination cluster and namespace
  destination:
    server: https://kubernetes.default.svc
    namespace: production

  # Sync policy
  syncPolicy:
    automated:
      prune: true # Delete resources not in Git
      selfHeal: true # Auto-sync if drift detected
    syncOptions:
      - CreateNamespace=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

### Multi-Environment Setup

```yaml
# Base
k8s/
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
└── overlays/
├── dev/
│   ├── kustomization.yaml
│   └── patches/
├── staging/
│   ├── kustomization.yaml
│   └── patches/
└── production/
├── kustomization.yaml
└── patches/
```

**Base Kustomization:**

```yaml
# k8s/base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
```

**Production Overlay:**

```yaml
# k8s/overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
bases:
  - ../../base
patchesStrategicMerge:
  - patches/replicas.yaml
namespace: production
images:
  - name: myapp
    newTag: v1.0.0
```

### ArgoCD CLI

```bash
# Install CLI
brew install argocd  # macOS
# or
curl -sSL -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64

# Login
argocd login localhost:8080

# Create application
argocd app create myapp \
  --repo https://github.com/myorg/myapp \
  --path k8s/overlays/production \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace production

# Sync application
argocd app sync myapp

# Get application status
argocd app get myapp

# List applications
argocd app list

# Delete application
argocd app delete myapp
```

---

## Flux CD

### Installation

```bash
# Install Flux CLI
brew install fluxcd/tap/flux  # macOS
# or
curl -s https://fluxcd.io/install.sh | sudo bash

# Check prerequisites
flux check --pre

# Bootstrap Flux (GitHub example)
export GITHUB_TOKEN=<your-token>
flux bootstrap github \
  --owner=myorg \
  --repository=fleet-infra \
  --branch=main \
  --path=./clusters/production \
  --personal
```

### Flux GitRepository

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: myapp
  namespace: flux-system
spec:
  interval: 1m
  url: https://github.com/myorg/myapp
  ref:
    branch: main
```

### Flux Kustomization

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: myapp
  namespace: flux-system
spec:
  interval: 5m
  path: ./k8s/overlays/production
  prune: true
  sourceRef:
    kind: GitRepository
    name: myapp
```

---

## Jenkins Pipeline

### Kubernetes Plugin

```groovy
// Jenkinsfile
pipeline {
    agent {
        kubernetes {
            yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: docker
    image: docker:latest
    command:
    - cat
    tty: true
    volumeMounts:
    - mountPath: /var/run/docker.sock
      name: docker-sock
  - name: kubectl
    image: bitnami/kubectl:latest
    command:
    - cat
    tty: true
  volumes:
  - name: docker-sock
    hostPath:
      path: /var/run/docker.sock
'''
        }
    }

    environment {
        DOCKER_REGISTRY = 'myregistry.azurecr.io'
        IMAGE_NAME = 'myapp'
        K8S_NAMESPACE = 'production'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                container('docker') {
                    sh '''
                        docker build -t ${DOCKER_REGISTRY}/${IMAGE_NAME}:${BUILD_NUMBER} .
                        docker tag ${DOCKER_REGISTRY}/${IMAGE_NAME}:${BUILD_NUMBER} ${DOCKER_REGISTRY}/${IMAGE_NAME}:latest
                    '''
                }
            }
        }

        stage('Push') {
            steps {
                container('docker') {
                    withCredentials([usernamePassword(credentialsId: 'docker-registry', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                        sh '''
                            echo $PASS | docker login ${DOCKER_REGISTRY} -u $USER --password-stdin
                            docker push ${DOCKER_REGISTRY}/${IMAGE_NAME}:${BUILD_NUMBER}
                            docker push ${DOCKER_REGISTRY}/${IMAGE_NAME}:latest
                        '''
                    }
                }
            }
        }

        stage('Deploy') {
            steps {
                container('kubectl') {
                    withKubeConfig([credentialsId: 'kubeconfig']) {
                        sh '''
                            kubectl set image deployment/myapp \
                              myapp=${DOCKER_REGISTRY}/${IMAGE_NAME}:${BUILD_NUMBER} \
                              -n ${K8S_NAMESPACE}
                            kubectl rollout status deployment/myapp -n ${K8S_NAMESPACE}
                        '''
                    }
                }
            }
        }
    }

    post {
        success {
            echo 'Deployment successful!'
        }
        failure {
            echo 'Deployment failed!'
        }
    }
}
```

---

## GitHub Actions

### Complete CI/CD Workflow

```yaml
# .github/workflows/deploy.yml
name: Build and Deploy to Kubernetes

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2

      - name: Log in to Container Registry
        uses: docker/login-action@v2
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v4
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=sha

      - name: Build and push image
        uses: docker/build-push-action@v4
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'

    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Configure kubectl
        uses: azure/k8s-set-context@v3
        with:
          method: kubeconfig
          kubeconfig: ${{ secrets.KUBE_CONFIG }}

      - name: Deploy to Kubernetes
        run: |
          # Update image tag in manifests
          sed -i "s|IMAGE_TAG|${{ github.sha }}|g" k8s/deployment.yaml

          # Apply manifests
          kubectl apply -f k8s/

          # Wait for rollout
          kubectl rollout status deployment/myapp -n production

      - name: Verify deployment
        run: |
          kubectl get pods -n production
          kubectl get services -n production
```

---

## Helm in CI/CD

### GitHub Actions with Helm

```yaml
name: Helm Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Configure kubectl
        uses: azure/k8s-set-context@v3
        with:
          method: kubeconfig
          kubeconfig: ${{ secrets.KUBE_CONFIG }}

      - name: Install Helm
        uses: azure/setup-helm@v3
        with:
          version: "v3.12.0"

      - name: Deploy with Helm
        run: |
          helm upgrade --install myapp ./helm/myapp \
            --namespace production \
            --create-namespace \
            --set image.tag=${{ github.sha }} \
            --set replicaCount=3 \
            --wait \
            --timeout 5m

      - name: Test deployment
        run: |
          helm test myapp -n production
```

---

## Deployment Strategies

### 1. Rolling Update (Default)

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
```

### 2. Blue-Green Deployment

```bash
# Deploy green (new version)
kubectl apply -f deployment-green.yaml

# Switch service to green
kubectl patch service myapp -p '{"spec":{"selector":{"version":"green"}}}'

# Verify, then delete blue
kubectl delete deployment myapp-blue
```

### 3. Canary Deployment

```yaml
# Stable deployment (90%)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-stable
spec:
  replicas: 9
  # ...
---
# Canary deployment (10%)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-canary
spec:
  replicas: 1
  # ... with new version
```

**Progressive Canary with Argo Rollouts:**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: myapp
spec:
  replicas: 10
  strategy:
    canary:
      steps:
        - setWeight: 10
        - pause: { duration: 10m }
        - setWeight: 20
        - pause: { duration: 10m }
        - setWeight: 50
        - pause: { duration: 10m }
        - setWeight: 100
  template:
    # ... pod spec
```

---

## Image Management

### Tagging Strategy

```bash
# Semantic versioning
myapp:1.2.3
myapp:1.2
myapp:1

# Git commit SHA
myapp:abc123def456

# Build number
myapp:build-1234

# Branch-based
myapp:main
myapp:develop

# Environment-based
myapp:production
myapp:staging

# Don't use 'latest' in production!
```

### Image Pull Policy

```yaml
spec:
  containers:
    - name: myapp
      image: myapp:1.2.3
      imagePullPolicy: IfNotPresent # or Always, Never
```

---

## Secrets in CI/CD

### Sealed Secrets

```bash
# Install Sealed Secrets controller
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.18.0/controller.yaml

# Install kubeseal CLI
brew install kubeseal

# Create sealed secret
echo -n 'supersecret' | kubectl create secret generic mysecret \
  --dry-run=client \
  --from-file=password=/dev/stdin \
  -o yaml | \
  kubeseal -o yaml > sealedsecret.yaml

# Commit sealedsecret.yaml to Git (safe!)
git add sealedsecret.yaml
git commit -m "Add sealed secret"
```

### External Secrets Operator

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: database-secret
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: db-credentials
    creationPolicy: Owner
  data:
    - secretKey: password
      remoteRef:
        key: database/prod
        property: password
```

---

## Testing in CI/CD

### Pre-deployment Tests

```yaml
# .github/workflows/test.yml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Run unit tests
        run: |
          make test

      - name: Run integration tests
        run: |
          docker-compose up -d
          make integration-test
          docker-compose down

      - name: Lint Kubernetes manifests
        run: |
          kubeval k8s/*.yaml

      - name: Security scan
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: "fs"
          scan-ref: "."
```

### Post-deployment Tests

```yaml
- name: Smoke tests
  run: |
    # Wait for deployment
    kubectl wait --for=condition=available deployment/myapp -n production --timeout=300s

    # Test endpoint
    ENDPOINT=$(kubectl get svc myapp -n production -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
    curl -f http://$ENDPOINT/health || exit 1

    # Run k6 load test
    k6 run tests/load-test.js
```

---

## Rollback Strategies

### Automatic Rollback

```yaml
# ArgoCD auto-rollback
apiVersion: argoproj.io/v1alpha1
kind: Application
spec:
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

### Manual Rollback

```bash
# Kubernetes rollback
kubectl rollout undo deployment/myapp -n production

# Rollback to specific revision
kubectl rollout history deployment/myapp -n production
kubectl rollout undo deployment/myapp --to-revision=2 -n production

# ArgoCD rollback
argocd app rollback myapp <history-id>
```

---

## Best Practices

### 1. Separate Concerns

```
separate repos:
├── app-repo/          (Application code)
│   ├── src/
│   ├── Dockerfile
│   └── .github/workflows/build.yml
└── config-repo/       (Kubernetes manifests)
    ├── k8s/
    └── .github/workflows/deploy.yml
```

### 2. Environment Parity

```yaml
# Use same base manifests for all environments
# Only vary:
# - Replica counts
# - Resource limits
# - Environment variables
# - Image tags
```

### 3. GitOps for Everything

```bash
✅ Store in Git:
- Kubernetes manifests
- Helm charts
- Kustomize overlays
- ArgoCD/Flux configurations

❌ Don't store in Git:
- Unencrypted secrets
- Binary files (use Git LFS)
- Generated files
```

### 4. Automated Testing

```yaml
Pipeline stages:
1. Lint code
2. Run unit tests
3. Build image
4. Scan image for vulnerabilities
5. Integration tests
6. Deploy to staging
7. Run smoke tests
8. Deploy to production
```

### 5. Progressive Rollout

```yaml
# Don't deploy all at once
1. Deploy to dev
2. Deploy canary to prod (10%)
3. Monitor metrics
4. Gradually increase (25%, 50%, 100%)
5. Auto-rollback if issues
```

---

## Key Takeaways

1. **GitOps** provides declarative, versioned deployments
2. **ArgoCD/Flux** automate Git-to-cluster sync
3. **Separate** application and configuration repos
4. **Use canary** or blue-green for production
5. **Automate testing** at every stage
6. **Encrypt secrets** before committing
7. **Tag images** with meaningful identifiers
8. **Monitor deployments** and set alerts
9. **Plan rollback** strategy
10. **Progressive rollout** reduces risk

---

## Next Steps

👉 **[14-best-practices.md](./14-best-practices.md)** - Production-ready Kubernetes best practices

## Additional Resources

- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [Flux CD Documentation](https://fluxcd.io/docs/)
- [Argo Rollouts](https://argoproj.github.io/argo-rollouts/)
- [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets)
- [External Secrets Operator](https://external-secrets.io/)
