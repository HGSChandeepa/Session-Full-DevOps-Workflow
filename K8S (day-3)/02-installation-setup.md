# Installation & Setup

## Overview

There are multiple ways to run Kubernetes depending on your needs. This guide covers four popular methods for getting started.

## Comparison of Installation Methods

| Distribution | Best For              | Pros                                | Cons                            |
| ------------ | --------------------- | ----------------------------------- | ------------------------------- |
| **K3s**      | Production, IoT, Edge | Lightweight, fast, production-ready | Fewer features than full K8s    |
| **Minikube** | Learning, Development | Easy to use, feature-complete       | Single node, resource intensive |
| **Kind**     | CI/CD, Testing        | Fast, runs in Docker                | Not for production              |
| **MicroK8s** | Development, IoT      | Easy snap installation              | Ubuntu/snap specific            |

## Method 1: K3s (Recommended for Beginners)

K3s is a lightweight Kubernetes distribution perfect for learning and production use. It bundles all components into a single binary.

### Prerequisites

- Linux, macOS, or Windows (WSL2)
- 1GB RAM minimum (2GB recommended)
- 512MB disk space

### Installation Steps

#### Linux/macOS

```bash
# Install K3s
curl -sfL https://get.k3s.io | sh -

# K3s will automatically start as a service
# Wait for the node to be ready (takes ~30 seconds)

# Create .kube directory
mkdir -p ~/.kube

# Copy kubeconfig file
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config

# Fix permissions
sudo chown $USER:$USER ~/.kube/config

# Set KUBECONFIG environment variable
export KUBECONFIG=~/.kube/config

# Add to shell profile for persistence
echo "export KUBECONFIG=~/.kube/config" >> ~/.bashrc  # or ~/.zshrc
```

#### Windows (WSL2)

```bash
# First, ensure WSL2 is installed
# Then run the same commands as Linux
curl -sfL https://get.k3s.io | sh -

# Follow the remaining Linux steps above
```

### Verify Installation

```bash
# Check cluster status
kubectl cluster-info

# Check nodes
kubectl get nodes

# Expected output:
# NAME     STATUS   ROLES                  AGE   VERSION
# node1    Ready    control-plane,master   1m    v1.27.x+k3s1
```

### Uninstall K3s

```bash
# Server
/usr/local/bin/k3s-uninstall.sh

# Agent (if you added additional nodes)
/usr/local/bin/k3s-agent-uninstall.sh
```

---

## Method 2: Minikube

Minikube creates a single-node Kubernetes cluster in a VM, perfect for learning all Kubernetes features.

### Prerequisites

- 2GB RAM minimum (4GB recommended)
- 20GB disk space
- Virtualization support (VT-x/AMD-v)
- Hypervisor (VirtualBox, Hyper-V, KVM) or Docker

### Installation

#### macOS

```bash
# Install using Homebrew
brew install minikube

# Or download binary
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-darwin-amd64
sudo install minikube-darwin-amd64 /usr/local/bin/minikube
```

#### Linux

```bash
# Download binary
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64

# Install
sudo install minikube-linux-amd64 /usr/local/bin/minikube
```

#### Windows

```powershell
# Using Chocolatey
choco install minikube

# Or download from GitHub releases
# https://github.com/kubernetes/minikube/releases
```

### Start Minikube

```bash
# Start with default settings (2GB RAM, 2 CPUs)
minikube start

# Start with custom resources
minikube start --memory=4096 --cpus=2

# Start with specific Kubernetes version
minikube start --kubernetes-version=v1.27.0

# Use Docker driver (no VM needed)
minikube start --driver=docker
```

### Verify Installation

```bash
# Check status
minikube status

# Get cluster info
kubectl cluster-info

# Access dashboard
minikube dashboard
```

### Useful Minikube Commands

```bash
# Stop cluster
minikube stop

# Delete cluster
minikube delete

# SSH into node
minikube ssh

# View logs
minikube logs

# Add-ons
minikube addons list
minikube addons enable ingress
minikube addons enable metrics-server
```

---

## Method 3: Kind (Kubernetes in Docker)

Kind runs Kubernetes clusters in Docker containers, making it extremely fast for testing.

### Prerequisites

- Docker installed and running
- 4GB RAM minimum

### Installation

```bash
# macOS/Linux
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind

# Or using Go
go install sigs.k8s.io/kind@v0.20.0

# Windows (PowerShell)
curl.exe -Lo kind-windows-amd64.exe https://kind.sigs.k8s.io/dl/v0.20.0/kind-windows-amd64
Move-Item .\kind-windows-amd64.exe c:\windows\system32\kind.exe
```

### Create Cluster

```bash
# Create single-node cluster
kind create cluster

# Create multi-node cluster
kind create cluster --config=kind-config.yaml
```

**kind-config.yaml** for multi-node cluster:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
  - role: worker
```

### Verify and Manage

```bash
# List clusters
kind get clusters

# Delete cluster
kind delete cluster

# Load Docker image into cluster
kind load docker-image my-app:latest
```

---

## Method 4: MicroK8s

MicroK8s is a lightweight Kubernetes for Ubuntu, providing a full Kubernetes experience with add-ons.

### Installation (Ubuntu)

```bash
# Install via snap
sudo snap install microk8s --classic

# Join user group
sudo usermod -a -G microk8s $USER
sudo chown -f -R $USER ~/.kube

# Refresh shell
newgrp microk8s
```

### Configure and Use

```bash
# Check status
microk8s status --wait-ready

# Enable add-ons
microk8s enable dns dashboard storage

# Use kubectl
microk8s kubectl get nodes

# Create alias (optional)
alias kubectl='microk8s kubectl'
```

---

## Installing kubectl

kubectl is the command-line tool for interacting with Kubernetes clusters.

### Linux

```bash
# Download latest
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

# Install
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Verify
kubectl version --client
```

### macOS

```bash
# Using Homebrew
brew install kubectl

# Or download binary
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/darwin/amd64/kubectl"
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl
```

### Windows

```powershell
# Using Chocolatey
choco install kubernetes-cli

# Or download from GitHub releases
```

---

## Post-Installation Setup

### Configure kubectl Autocompletion

**Bash:**

```bash
echo 'source <(kubectl completion bash)' >>~/.bashrc
```

**Zsh:**

```bash
echo 'source <(kubectl completion zsh)' >>~/.zshrc
```

### Create kubectl Alias

```bash
# Add to ~/.bashrc or ~/.zshrc
alias k=kubectl
complete -F __start_kubectl k
```

### Verify Everything Works

```bash
# Check cluster
kubectl cluster-info

# Check nodes
kubectl get nodes

# Check system pods
kubectl get pods -n kube-system

# Create test deployment
kubectl create deployment nginx --image=nginx

# Check deployment
kubectl get deployments
kubectl get pods

# Clean up
kubectl delete deployment nginx
```

---

## Managed Kubernetes Services (Cloud)

For production workloads, consider managed services:

### Amazon EKS (AWS)

```bash
# Install eksctl
brew install weaveworks/tap/eksctl  # macOS

# Create cluster
eksctl create cluster --name my-cluster --region us-west-2
```

### Google GKE (Google Cloud)

```bash
# Install gcloud SDK
# Then create cluster
gcloud container clusters create my-cluster --num-nodes=3
```

### Azure AKS (Microsoft Azure)

```bash
# Install Azure CLI
# Then create cluster
az aks create --resource-group myResourceGroup --name myAKSCluster --node-count 3
```

---

## Troubleshooting Common Issues

### K3s Issues

```bash
# Check service status
sudo systemctl status k3s

# View logs
sudo journalctl -u k3s -f

# Restart service
sudo systemctl restart k3s
```

### Minikube Issues

```bash
# If cluster won't start
minikube delete
minikube start --driver=docker

# Check Docker
docker ps

# Increase resources
minikube start --memory=8192 --cpus=4
```

### kubectl Connection Issues

```bash
# Check config
kubectl config view

# Check current context
kubectl config current-context

# Switch context
kubectl config use-context minikube
```

---

## Next Steps

Now that you have Kubernetes installed:

👉 **[03-core-concepts.md](./03-core-concepts.md)** - Learn fundamental Kubernetes concepts

## Quick Reference

| Task          | Command                             |
| ------------- | ----------------------------------- |
| Check cluster | `kubectl cluster-info`              |
| View nodes    | `kubectl get nodes`                 |
| View pods     | `kubectl get pods --all-namespaces` |
| Check version | `kubectl version`                   |

## Additional Resources

- [K3s Documentation](https://docs.k3s.io/)
- [Minikube Documentation](https://minikube.sigs.k8s.io/docs/)
- [Kind Documentation](https://kind.sigs.k8s.io/)
- [kubectl Installation Guide](https://kubernetes.io/docs/tasks/tools/)
