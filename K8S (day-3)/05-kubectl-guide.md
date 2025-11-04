# Working with kubectl

## Overview

`kubectl` (Kube Control) is the command-line tool for interacting with Kubernetes clusters. Mastering kubectl is essential for working with Kubernetes effectively.

## kubectl Syntax

```bash
kubectl [command] [TYPE] [NAME] [flags]
```

- **command**: Operation (get, create, describe, delete, etc.)
- **TYPE**: Resource type (pod, service, deployment, etc.)
- **NAME**: Resource name (optional)
- **flags**: Additional options

**Examples:**

```bash
kubectl get pods
kubectl describe pod nginx
kubectl delete deployment webapp
kubectl logs pod/nginx -f
```

---

## Essential Commands

### Resource Management

#### Create Resources

```bash
# Create from file
kubectl create -f <filename>
kubectl apply -f <filename>

# Create from multiple files
kubectl apply -f ./dir/
kubectl apply -f file1.yaml -f file2.yaml

# Create from URL
kubectl apply -f https://example.com/manifest.yaml

# Create specific resources
kubectl create deployment nginx --image=nginx
kubectl create service clusterip my-svc --tcp=80:80
kubectl create namespace dev
kubectl create configmap my-config --from-literal=key1=value1
kubectl create secret generic my-secret --from-literal=password=secret123
```

**create vs apply:**

- `create`: Imperative (creates new resources, errors if exists)
- `apply`: Declarative (creates or updates, idempotent)

#### View Resources

```bash
# List resources
kubectl get <resource>
kubectl get pods
kubectl get deployments
kubectl get services
kubectl get all  # Most common resources

# List from all namespaces
kubectl get pods --all-namespaces
kubectl get pods -A  # Short form

# List from specific namespace
kubectl get pods -n kube-system

# Show labels
kubectl get pods --show-labels

# Wide output (more columns)
kubectl get pods -o wide

# Watch for changes
kubectl get pods --watch
kubectl get pods -w  # Short form

# Custom columns
kubectl get pods -o custom-columns=NAME:.metadata.name,STATUS:.status.phase

# JSON output
kubectl get pod nginx -o json

# YAML output
kubectl get pod nginx -o yaml

# Get multiple resource types
kubectl get pods,services,deployments
```

#### Describe Resources

```bash
# Detailed information
kubectl describe <resource> <name>
kubectl describe pod nginx
kubectl describe node minikube
kubectl describe deployment webapp

# Describe all pods
kubectl describe pods
```

#### Update Resources

```bash
# Edit resource interactively
kubectl edit <resource> <name>
kubectl edit deployment nginx

# Update image
kubectl set image deployment/nginx nginx=nginx:1.22

# Scale deployment
kubectl scale deployment nginx --replicas=5

# Update resource from file
kubectl apply -f updated-deployment.yaml

# Patch resource
kubectl patch deployment nginx -p '{"spec":{"replicas":3}}'
```

#### Delete Resources

```bash
# Delete specific resource
kubectl delete pod nginx
kubectl delete deployment webapp

# Delete from file
kubectl delete -f deployment.yaml

# Delete all pods
kubectl delete pods --all

# Delete with label selector
kubectl delete pods -l app=nginx

# Delete namespace (deletes all resources in it)
kubectl delete namespace dev

# Force delete (use cautiously)
kubectl delete pod nginx --force --grace-period=0
```

---

### Pod Operations

#### Running Pods

```bash
# Run simple pod
kubectl run nginx --image=nginx

# Run with specific port
kubectl run nginx --image=nginx --port=80

# Run with environment variables
kubectl run nginx --image=nginx --env="ENV=prod"

# Run with command
kubectl run busybox --image=busybox -- sleep 3600

# Run interactive pod
kubectl run -it busybox --image=busybox --restart=Never -- sh

# Run and delete after exit
kubectl run temp --image=nginx --rm -it -- sh
```

#### Accessing Pods

```bash
# View logs
kubectl logs nginx
kubectl logs nginx -f  # Follow logs
kubectl logs nginx --tail=50  # Last 50 lines
kubectl logs nginx --since=1h  # Logs from last hour
kubectl logs nginx -c container-name  # Specific container in pod

# Execute command in pod
kubectl exec nginx -- ls /
kubectl exec nginx -- cat /etc/nginx/nginx.conf

# Interactive shell
kubectl exec -it nginx -- /bin/bash
kubectl exec -it nginx -- sh

# Copy files to/from pod
kubectl cp local-file.txt nginx:/tmp/file.txt
kubectl cp nginx:/etc/nginx/nginx.conf ./nginx.conf
```

#### Port Forwarding

```bash
# Forward local port to pod
kubectl port-forward pod/nginx 8080:80

# Forward to service
kubectl port-forward service/nginx 8080:80

# Forward to deployment
kubectl port-forward deployment/nginx 8080:80

# Listen on specific address
kubectl port-forward --address 0.0.0.0 pod/nginx 8080:80
```

---

### Deployment Operations

```bash
# Create deployment
kubectl create deployment nginx --image=nginx --replicas=3

# View deployments
kubectl get deployments

# View deployment details
kubectl describe deployment nginx

# Scale deployment
kubectl scale deployment nginx --replicas=5

# Autoscale deployment
kubectl autoscale deployment nginx --min=2 --max=10 --cpu-percent=80

# Update deployment image (rolling update)
kubectl set image deployment/nginx nginx=nginx:1.22

# Check rollout status
kubectl rollout status deployment/nginx

# Pause rollout
kubectl rollout pause deployment/nginx

# Resume rollout
kubectl rollout resume deployment/nginx

# View rollout history
kubectl rollout history deployment/nginx

# Rollback to previous version
kubectl rollout undo deployment/nginx

# Rollback to specific revision
kubectl rollout undo deployment/nginx --to-revision=2

# Restart deployment (recreate pods)
kubectl rollout restart deployment/nginx
```

---

### Service Operations

```bash
# Create service
kubectl create service clusterip my-svc --tcp=80:80

# Expose deployment as service
kubectl expose deployment nginx --port=80 --target-port=80

# Expose with LoadBalancer
kubectl expose deployment nginx --type=LoadBalancer --port=80

# Expose with NodePort
kubectl expose deployment nginx --type=NodePort --port=80

# View services
kubectl get services
kubectl get svc  # Short form

# Describe service
kubectl describe service nginx

# Delete service
kubectl delete service nginx
```

---

### Configuration Management

#### ConfigMaps

```bash
# Create from literal
kubectl create configmap my-config \
  --from-literal=key1=value1 \
  --from-literal=key2=value2

# Create from file
kubectl create configmap my-config --from-file=config.txt

# Create from directory
kubectl create configmap my-config --from-file=./configs/

# View configmaps
kubectl get configmaps
kubectl get cm  # Short form

# Describe configmap
kubectl describe configmap my-config

# View configmap data
kubectl get configmap my-config -o yaml
```

#### Secrets

```bash
# Create generic secret
kubectl create secret generic my-secret \
  --from-literal=username=admin \
  --from-literal=password=secret123

# Create from file
kubectl create secret generic my-secret --from-file=ssh-key=~/.ssh/id_rsa

# Create TLS secret
kubectl create secret tls tls-secret \
  --cert=path/to/cert.crt \
  --key=path/to/key.key

# Create docker registry secret
kubectl create secret docker-registry my-registry \
  --docker-server=registry.example.com \
  --docker-username=user \
  --docker-password=pass \
  --docker-email=user@example.com

# View secrets (values encoded)
kubectl get secrets

# Decode secret value
kubectl get secret my-secret -o jsonpath='{.data.password}' | base64 --decode
```

---

### Namespace Operations

```bash
# List namespaces
kubectl get namespaces
kubectl get ns  # Short form

# Create namespace
kubectl create namespace dev

# Delete namespace
kubectl delete namespace dev

# Set default namespace
kubectl config set-context --current --namespace=dev

# View current namespace
kubectl config view --minify | grep namespace:

# Run command in specific namespace
kubectl get pods -n kube-system

# Get resources from all namespaces
kubectl get pods --all-namespaces
kubectl get pods -A  # Short form
```

---

### Label and Annotation Operations

```bash
# Show labels
kubectl get pods --show-labels

# Add label
kubectl label pod nginx env=prod

# Update label
kubectl label pod nginx env=staging --overwrite

# Remove label
kubectl label pod nginx env-

# Select by label
kubectl get pods -l env=prod
kubectl get pods -l 'env in (prod,staging)'
kubectl get pods -l env!=dev

# Add annotation
kubectl annotate pod nginx description="Web server"

# View annotations
kubectl describe pod nginx | grep Annotations -A 5
```

---

### Context and Configuration

```bash
# View config
kubectl config view

# View current context
kubectl config current-context

# List contexts
kubectl config get-contexts

# Switch context
kubectl config use-context minikube

# Set namespace for context
kubectl config set-context --current --namespace=dev

# Add new cluster
kubectl config set-cluster my-cluster --server=https://1.2.3.4:6443

# Add user credentials
kubectl config set-credentials my-user --token=bearer_token

# Create new context
kubectl config set-context my-context --cluster=my-cluster --user=my-user

# Delete context
kubectl config delete-context old-context
```

---

### Troubleshooting Commands

```bash
# Get events
kubectl get events
kubectl get events --sort-by='.lastTimestamp'
kubectl get events -n kube-system

# View logs
kubectl logs nginx
kubectl logs nginx --previous  # Previous container instance

# Describe for troubleshooting
kubectl describe pod nginx

# Check resource usage
kubectl top nodes
kubectl top pods

# Debug with ephemeral container
kubectl debug nginx -it --image=busybox

# Get pod manifest
kubectl get pod nginx -o yaml

# Explain resource
kubectl explain pods
kubectl explain pods.spec.containers
```

---

### Advanced Operations

#### Diff and Apply

```bash
# Preview changes before applying
kubectl diff -f deployment.yaml

# Apply changes
kubectl apply -f deployment.yaml

# Server-side apply
kubectl apply -f deployment.yaml --server-side
```

#### Dry Run

```bash
# Test without creating
kubectl create deployment nginx --image=nginx --dry-run=client -o yaml

# Generate YAML
kubectl create deployment nginx --image=nginx --dry-run=client -o yaml > deployment.yaml
```

#### JSONPath Queries

```bash
# Get specific field
kubectl get pods -o jsonpath='{.items[0].metadata.name}'

# Get all pod names
kubectl get pods -o jsonpath='{.items[*].metadata.name}'

# Complex query
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.phase}{"\n"}{end}'

# Get node IPs
kubectl get nodes -o jsonpath='{.items[*].status.addresses[?(@.type=="InternalIP")].address}'
```

#### Resource Quotas and Limits

```bash
# View resource quotas
kubectl get resourcequotas
kubectl describe resourcequota

# View limit ranges
kubectl get limitranges
kubectl describe limitrange
```

---

## Output Formats

```bash
# Wide output
kubectl get pods -o wide

# YAML
kubectl get pod nginx -o yaml

# JSON
kubectl get pod nginx -o json

# Custom columns
kubectl get pods -o custom-columns=NAME:.metadata.name,STATUS:.status.phase

# JSONPath
kubectl get pods -o jsonpath='{.items[*].metadata.name}'

# Name only
kubectl get pods -o name

# Go template
kubectl get pods -o go-template='{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}'
```

---

## Useful Aliases

Add these to your `~/.bashrc` or `~/.zshrc`:

```bash
# Basic aliases
alias k='kubectl'
alias kg='kubectl get'
alias kd='kubectl describe'
alias kdel='kubectl delete'
alias kl='kubectl logs'
alias kex='kubectl exec -it'

# Context aliases
alias kctx='kubectl config current-context'
alias kns='kubectl config set-context --current --namespace'

# Common operations
alias kgp='kubectl get pods'
alias kgs='kubectl get services'
alias kgd='kubectl get deployments'
alias kgn='kubectl get nodes'

# All namespaces
alias kgpa='kubectl get pods --all-namespaces'

# Apply/Delete
alias ka='kubectl apply -f'
alias kdelf='kubectl delete -f'

# Logs
alias klf='kubectl logs -f'
```

Enable bash completion:

```bash
source <(kubectl completion bash)
complete -F __start_kubectl k  # If using alias k=kubectl
```

---

## kubectl Cheat Sheet

### Quick Reference Table

| Task              | Command                                            |
| ----------------- | -------------------------------------------------- |
| Get pods          | `kubectl get pods`                                 |
| Describe pod      | `kubectl describe pod <name>`                      |
| Delete pod        | `kubectl delete pod <name>`                        |
| Pod logs          | `kubectl logs <pod>`                               |
| Execute in pod    | `kubectl exec -it <pod> -- bash`                   |
| Create deployment | `kubectl create deployment <name> --image=<image>` |
| Scale deployment  | `kubectl scale deployment <name> --replicas=N`     |
| Expose deployment | `kubectl expose deployment <name> --port=80`       |
| Apply YAML        | `kubectl apply -f <file>`                          |
| Get services      | `kubectl get services`                             |
| Port forward      | `kubectl port-forward <pod> 8080:80`               |
| View events       | `kubectl get events`                               |
| Top nodes         | `kubectl top nodes`                                |
| Cluster info      | `kubectl cluster-info`                             |

---

## Practice Exercises

### Exercise 1: Pod Management

```bash
# Create a pod
kubectl run nginx --image=nginx

# Check status
kubectl get pods

# View logs
kubectl logs nginx

# Execute command
kubectl exec nginx -- nginx -v

# Get shell
kubectl exec -it nginx -- /bin/bash

# Delete pod
kubectl delete pod nginx
```

### Exercise 2: Deployment Workflow

```bash
# Create deployment
kubectl create deployment web --image=nginx --replicas=3

# Scale up
kubectl scale deployment web --replicas=5

# Update image
kubectl set image deployment/web nginx=nginx:alpine

# Check rollout
kubectl rollout status deployment/web

# View history
kubectl rollout history deployment/web

# Rollback
kubectl rollout undo deployment/web

# Clean up
kubectl delete deployment web
```

### Exercise 3: Service Discovery

```bash
# Create deployment
kubectl create deployment nginx --image=nginx

# Expose as service
kubectl expose deployment nginx --port=80

# Test with temporary pod
kubectl run test --image=busybox --rm -it --restart=Never -- wget -qO- http://nginx

# Port forward to local
kubectl port-forward service/nginx 8080:80

# Clean up
kubectl delete deployment nginx
kubectl delete service nginx
```

---

## Key Takeaways

1. **kubectl** is your primary interface to Kubernetes
2. Use **apply** for declarative, **create** for imperative management
3. Master **get**, **describe**, **logs**, and **exec** for troubleshooting
4. Learn **port-forward** for local testing
5. Use **-o yaml** and **--dry-run** to generate manifests
6. Leverage **aliases** and **completion** for efficiency

---

## Next Steps

👉 **[06-deployments-replicasets.md](./06-deployments-replicasets.md)** - Deep dive into Deployments and ReplicaSets

## Additional Resources

- [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
- [kubectl Reference](https://kubernetes.io/docs/reference/kubectl/)
- [kubectl Book](https://kubectl.docs.kubernetes.io/)
