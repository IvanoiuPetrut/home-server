# Home Server - Kubernetes on Raspberry Pi

A self-hosted media server running on a Raspberry Pi with K3s (lightweight Kubernetes). This project is designed for learning Kubernetes concepts while building something practical.

## 📚 Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Kubernetes Concepts Explained](#kubernetes-concepts-explained)
- [Project Structure](#project-structure)
- [File-by-File Breakdown](#file-by-file-breakdown)
- [Prerequisites](#prerequisites)
- [Setup Guide](#setup-guide)
- [Deployment](#deployment)
- [Troubleshooting](#troubleshooting)
- [Learning Resources](#learning-resources)

---

## Overview

### What is This Project?

This project deploys a **Plex Media Server** on a Raspberry Pi using **K3s**, a lightweight Kubernetes distribution. Plex allows you to organize and stream your personal media (movies, TV shows, music) to any device on your network or remotely.

### Why Kubernetes on a Raspberry Pi?

- **Learning**: Kubernetes is the industry standard for container orchestration. Running it at home gives you hands-on experience.
- **Resilience**: Kubernetes automatically restarts crashed containers and manages resources.
- **Scalability**: Easy to add more services (Sonarr, Radarr, etc.) using the same patterns.
- **GitOps Ready**: All configuration is in YAML files, perfect for version control.

### What is K3s?

K3s is a certified Kubernetes distribution designed for:

- **Resource-constrained environments** (like Raspberry Pi)
- **Edge computing** and IoT
- **Single-node clusters** (though it supports multi-node too)

It removes heavy components (like etcd, cloud providers) and packages everything into a single binary under 100MB.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                      Raspberry Pi (Ubuntu Server)               │
├─────────────────────────────────────────────────────────────────┤
│                           K3s Cluster                           │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                    Namespace: media                       │  │
│  │  ┌─────────────────────────────────────────────────────┐  │  │
│  │  │              Deployment: plex                       │  │  │
│  │  │  ┌───────────────────────────────────────────────┐  │  │  │
│  │  │  │                 Pod: plex                     │  │  │  │
│  │  │  │  ┌─────────────────────────────────────────┐  │  │  │  │
│  │  │  │  │      Container: linuxserver/plex        │  │  │  │  │
│  │  │  │  │                                         │  │  │  │  │
│  │  │  │  │   Port 32400 ◄─── Service (LoadBalancer)│  │  │  │  │
│  │  │  │  └─────────────────────────────────────────┘  │  │  │  │
│  │  │  │       │              │              │         │  │  │  │
│  │  │  │       ▼              ▼              ▼         │  │  │  │
│  │  │  │   /config        /data         /transcode    │  │  │  │
│  │  │  └───────┬──────────────┬──────────────┬────────┘  │  │  │
│  │  └──────────┼──────────────┼──────────────┼───────────┘  │  │
│  └─────────────┼──────────────┼──────────────┼──────────────┘  │
│                │              │              │                  │
│                ▼              ▼              ▼                  │
│         PVC: config    PVC: media      emptyDir (RAM)          │
│                │              │                                 │
│                ▼              ▼                                 │
│         PV: config     PV: media                               │
│                │              │                                 │
└────────────────┼──────────────┼─────────────────────────────────┘
                 │              │
                 ▼              ▼
    ┌────────────────────────────────────────┐
    │         External HDD (USB)             │
    │  /mnt/external-hdd/plex/config         │
    │  /mnt/external-hdd/media               │
    └────────────────────────────────────────┘
```

---

## Kubernetes Concepts Explained

### 🏷️ Namespace

**What**: A virtual cluster within your physical cluster. Think of it as a folder that groups related resources.

**Why**:

- Organizes resources logically (all media apps in `media` namespace)
- Allows resource quotas per namespace
- Provides scope for names (you can have a `plex` service in multiple namespaces)

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: media # All our media apps live here
```

### 📦 Pod

**What**: The smallest deployable unit in Kubernetes. A pod wraps one or more containers that share storage and network.

**Why**: Containers in a pod can communicate via `localhost` and share volumes. Usually, one container per pod.

### 🚀 Deployment

**What**: Manages pods declaratively. You describe the desired state, and Kubernetes makes it happen.

**Why**:

- **Self-healing**: If a pod crashes, the Deployment creates a new one
- **Rolling updates**: Update your app without downtime
- **Scaling**: Change `replicas` to run multiple instances

**Key Fields**:

```yaml
spec:
  replicas: 1 # How many pod copies to run
  selector: # How to find pods this Deployment manages
    matchLabels:
      app: plex
  strategy:
    type: Recreate # Kill old pod before creating new (needed for Plex)
  template: # Pod template - what each pod looks like
    spec:
      containers: [...]
```

### 💾 PersistentVolume (PV) & PersistentVolumeClaim (PVC)

**What**:

- **PV**: A piece of storage in the cluster (like a hard drive)
- **PVC**: A request for storage by a pod (like asking for a hard drive)

**Why**: Containers are ephemeral - when they restart, data is lost. PVs provide persistent storage that survives pod restarts.

**The Relationship**:

```
Pod → PVC → PV → Actual Storage (your external HDD)
     "I need       "Here's        "Points to
      10Gi"         10Gi"          /mnt/external-hdd"
```

**Access Modes**:

- `ReadWriteOnce (RWO)`: One pod can read/write
- `ReadOnlyMany (ROX)`: Many pods can read
- `ReadWriteMany (RWX)`: Many pods can read/write

### 🌐 Service

**What**: An abstract way to expose pods to network traffic. Provides a stable IP/DNS name for a set of pods.

**Why**: Pods are ephemeral and get new IPs when recreated. Services provide a stable endpoint.

**Types**:

- `ClusterIP`: Internal only (default)
- `NodePort`: Exposes on each node's IP at a static port
- `LoadBalancer`: Exposes externally using a load balancer (K3s uses ServiceLB)

### 🔐 Secret

**What**: Stores sensitive data like passwords, tokens, and keys.

**Why**:

- Keeps secrets out of your container images
- Can be mounted as files or environment variables
- Should never be committed to Git!

### 📋 Kustomize

**What**: A tool built into `kubectl` for customizing Kubernetes configurations without templates.

**Why**:

- Combine multiple YAML files into one deployment
- Override values for different environments (dev/prod)
- No need for Helm's complexity for simple projects

---

## Project Structure

```
home-server/
├── .gitignore                    # Prevents secrets from being committed
├── README.md                     # This file
└── cluster/                      # All Kubernetes manifests
    ├── namespace.yaml            # Creates the 'media' namespace (kept for reference)
    └── plex/                     # Plex-specific resources
        ├── kustomization.yaml    # Kustomize configuration
        ├── namespace.yaml        # Namespace definition
        ├── deployment.yaml       # Plex container configuration
        ├── service.yaml          # Network exposure
        ├── pvc.yaml              # Storage configuration
        └── secret.yaml           # Secrets (uses env variable substitution)
```

---

## File-by-File Breakdown

### `cluster/namespace.yaml`

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: media
  labels:
    app.kubernetes.io/name: media
```

**Purpose**: Creates an isolated namespace for all media-related applications.

**Fields Explained**:

- `apiVersion: v1` - Core Kubernetes API (namespaces are fundamental)
- `kind: Namespace` - The type of resource we're creating
- `metadata.name` - The namespace name, used in other resources
- `metadata.labels` - Key-value pairs for organizing/selecting resources

---

### `cluster/plex/deployment.yaml`

This is the most complex file. Let's break it down section by section:

#### Metadata Section

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: plex
  namespace: media
  labels:
    app: plex
```

- `apiVersion: apps/v1` - Deployments are in the "apps" API group
- `namespace: media` - Deploy into our media namespace
- `labels` - Tags for identifying this deployment

#### Spec Section

```yaml
spec:
  replicas: 1
  selector:
    matchLabels:
      app: plex
  strategy:
    type: Recreate
```

- `replicas: 1` - Only one Plex instance (it doesn't support clustering)
- `selector` - How the Deployment finds its pods (must match pod labels)
- `strategy: Recreate` - Stop old pod before starting new one (Plex needs exclusive access to its database)

#### Pod Template - Container

```yaml
spec:
  hostNetwork: true
  containers:
    - name: plex
      image: linuxserver/plex:arm64v8-latest
      imagePullPolicy: IfNotPresent
```

- `hostNetwork: true` - **Critical for Plex!** Uses the host's network directly, enabling:
  - DLNA discovery
  - Local network detection
  - GDM (G'Day Mate) protocol for finding other Plex servers
- `image` - LinuxServer.io's ARM64 Plex image (optimized for Raspberry Pi)
- `imagePullPolicy: IfNotPresent` - Don't re-download if image exists locally

#### Environment Variables

```yaml
env:
  - name: TZ
    value: "Europe/Bucharest"
  - name: PUID
    value: "1000"
  - name: PGID
    value: "1000"
  - name: PLEX_CLAIM
    valueFrom:
      secretKeyRef:
        name: plex-secrets
        key: claim-token
        optional: true
```

- `TZ` - Timezone for correct timestamps in Plex
- `PUID/PGID` - User/Group ID the container runs as (should match your file ownership)
- `PLEX_CLAIM` - Token to link Plex to your account (pulled from Secret, optional so it doesn't fail if missing)

#### Volume Mounts

```yaml
volumeMounts:
  - name: config
    mountPath: /config
  - name: media
    mountPath: /data
    readOnly: true
  - name: transcode
    mountPath: /transcode
```

- `/config` - Plex database, metadata, settings (persistent)
- `/data` - Your media files (read-only for safety)
- `/transcode` - Temporary transcoding files (in RAM for speed)

#### Resource Limits

```yaml
resources:
  requests:
    memory: "512Mi"
    cpu: "500m"
  limits:
    memory: "2Gi"
    cpu: "2000m"
```

- `requests` - Minimum guaranteed resources
- `limits` - Maximum allowed resources
- `500m` = 0.5 CPU cores, `2000m` = 2 CPU cores
- Prevents Plex from consuming all Pi resources

#### Health Probes

```yaml
livenessProbe:
  httpGet:
    path: /identity
    port: 32400
  initialDelaySeconds: 60
  periodSeconds: 30
readinessProbe:
  httpGet:
    path: /identity
    port: 32400
  initialDelaySeconds: 30
  periodSeconds: 10
```

- `livenessProbe` - "Is the container alive?" If it fails, Kubernetes restarts the container
- `readinessProbe` - "Is the container ready for traffic?" If it fails, traffic stops being sent
- `/identity` - Plex endpoint that returns server info (lightweight health check)

#### Volumes Definition

```yaml
volumes:
  - name: config
    persistentVolumeClaim:
      claimName: plex-config-pvc
  - name: media
    persistentVolumeClaim:
      claimName: plex-media-pvc
  - name: transcode
    emptyDir:
      medium: Memory
      sizeLimit: 1Gi
```

- `persistentVolumeClaim` - References our PVCs for persistent storage
- `emptyDir` with `medium: Memory` - Creates a RAM disk (fast but temporary)

---

### `cluster/plex/pvc.yaml`

Contains 4 resources: 2 PersistentVolumes and 2 PersistentVolumeClaims.

#### Config PV/PVC

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: plex-config-pv
spec:
  storageClassName: manual
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /mnt/external-hdd/plex/config
```

- `storageClassName: manual` - We're manually provisioning storage (not using a dynamic provisioner)
- `capacity` - How much storage this PV provides
- `accessModes: ReadWriteOnce` - Only one pod can mount this read-write
- `hostPath` - Points to actual directory on the host filesystem

#### Media PV/PVC

```yaml
accessModes:
  - ReadOnlyMany
hostPath:
  path: /mnt/external-hdd/media
```

- `ReadOnlyMany` - Multiple pods could read this (useful if you add more apps later)
- Points to your media library

---

### `cluster/plex/service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: plex
  namespace: media
spec:
  type: LoadBalancer
  ports:
    - name: pms
      port: 32400
      targetPort: 32400
      protocol: TCP
  selector:
    app: plex
```

**Purpose**: Exposes Plex to your network.

- `type: LoadBalancer` - K3s's ServiceLB assigns an external IP
- `port: 32400` - The port clients connect to
- `targetPort: 32400` - The port on the container
- `selector` - Routes traffic to pods with label `app: plex`

> **Note**: With `hostNetwork: true`, the service is somewhat redundant since Plex binds directly to the host's port 32400. However, it's good practice and useful for service discovery within the cluster.

---

### `cluster/plex/secret.yaml`

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: plex-secrets
  namespace: media
type: Opaque
stringData:
  claim-token: "${PLEX_CLAIM_TOKEN}"
```

**Purpose**: Stores your Plex claim token securely using environment variable substitution.

- `type: Opaque` - Generic secret type
- `stringData` - Kubernetes will base64-encode this automatically
- `${PLEX_CLAIM_TOKEN}` - Placeholder that gets replaced with your actual token at deploy time

**How It Works**:

The `${PLEX_CLAIM_TOKEN}` placeholder is substituted using `envsubst` before applying to Kubernetes. This means:

- The secret file can be safely committed to Git (it only contains the placeholder)
- The actual token is set as an environment variable on your Raspberry Pi
- No need to create or edit files on the server

---

### `cluster/plex/kustomization.yaml`

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: media

resources:
  - ../namespace.yaml
  - pvc.yaml
  - deployment.yaml
  - service.yaml
  - secret.yaml
```

**Purpose**: Combines all resources for single-command deployment.

- `namespace: media` - Applies this namespace to all resources
- `resources` - List of files to include (order matters for dependencies)

---

### `.gitignore`

```
.env
.env.*
!.env.example
```

**Purpose**: Prevents local environment files from being committed to Git.

---

## Prerequisites

### Hardware

- Raspberry Pi 4 (2GB+ RAM recommended)
- External HDD connected via USB
- Network connection (Ethernet recommended for stability)

### Software

- Ubuntu Server (64-bit) installed on the external HDD
- K3s installed and running

### Install K3s

```bash
# Install K3s (single node)
curl -sfL https://get.k3s.io | sh -

# Verify installation
sudo kubectl get nodes

# Make kubectl work without sudo (optional)
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $USER:$USER ~/.kube/config
export KUBECONFIG=~/.kube/config
```

---

## Setup Guide

### 1. Clone the Repository on Your Raspberry Pi

SSH into your Raspberry Pi and clone the project:

```bash
# SSH into your Raspberry Pi (from your computer)
ssh your-username@raspberry-pi-ip

# Install git if not already installed
sudo apt update && sudo apt install -y git

# Choose where to store your configs (home directory is fine)
cd ~

# Clone your repository (replace with your GitHub username)
git clone https://github.com/YOUR_USERNAME/home-server.git

# Enter the project directory
cd home-server
```

**Recommended location**: `~/home-server` or `/opt/home-server`

> **Tip**: If you want to pull updates later, just run `git pull` from the project directory.

---

### 2. Prepare Storage Directories

First, find your external HDD mount point:

```bash
# List all mounted drives
lsblk

# Or check disk usage to find your external HDD
df -h
```

Your external HDD is likely mounted at something like:

- `/mnt/external-hdd`
- `/media/your-username/drive-name`
- `/mnt/usb`

Then create the directories:

```bash
# Replace /mnt/external-hdd with your actual mount point!
sudo mkdir -p /mnt/external-hdd/plex/config
sudo mkdir -p /mnt/external-hdd/media

# Set ownership (1000 is typically the first user)
sudo chown -R 1000:1000 /mnt/external-hdd/plex

# IMPORTANT: Update the paths in pvc.yaml to match your mount point
nano ~/home-server/cluster/plex/pvc.yaml
# Change /mnt/external-hdd to your actual path
```

### 3. Add Your Media

```bash
# Copy or move your media files
cp -r /path/to/your/movies /mnt/external-hdd/media/movies
cp -r /path/to/your/tvshows /mnt/external-hdd/media/tvshows
```

### 4. Set Up Environment Variable for Plex Claim Token

The Plex claim token links your server to your Plex account. It's optional but recommended.

1. Go to https://www.plex.tv/claim/
2. Copy the claim token (valid for 4 minutes!)
3. Set it as an environment variable on your Raspberry Pi:

```bash
# For current session
export PLEX_CLAIM_TOKEN="claim-xxxxxxxxxxxxxxxxxxxx"

# To persist across reboots, add to your shell profile
echo 'export PLEX_CLAIM_TOKEN="claim-xxxxxxxxxxxxxxxxxxxx"' >> ~/.bashrc
source ~/.bashrc
```

> **Note**: The claim token is only needed for the initial setup. After Plex is linked to your account, you can remove the environment variable.

---

## Deployment

### Using Kustomize (Recommended)

```bash
# First, set up kubectl permissions (one-time setup)
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $USER:$USER ~/.kube/config
export KUBECONFIG=~/.kube/config
echo 'export KUBECONFIG=~/.kube/config' >> ~/.bashrc

# Navigate to your project
cd ~/home-server

# Set your Plex claim token
export PLEX_CLAIM_TOKEN="claim-xxxxxxxxxxxxxxxxxxxx"

# Deploy with environment variable substitution
# Note: K3s includes kustomize in kubectl, so use 'kubectl kustomize'
kubectl kustomize cluster/plex/ | envsubst | kubectl apply -f -

# Watch the deployment
kubectl -n media get pods -w

# Check all resources
kubectl -n media get all
```

**What's happening here?**

1. `kubectl kustomize cluster/plex/` - Combines all YAML files (K3s has kustomize built-in)
2. `envsubst` - Replaces `${PLEX_CLAIM_TOKEN}` with your actual token
3. `kubectl apply -f -` - Applies the result to Kubernetes

### Manual Deployment

```bash
# Set up kubectl permissions (if not done already)
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $USER:$USER ~/.kube/config
export KUBECONFIG=~/.kube/config

# Set your environment variable
export PLEX_CLAIM_TOKEN="claim-xxxxxxxxxxxxxxxxxxxx"

# Navigate to project
cd ~/home-server

# Apply in order (namespace first, then storage, then app)
kubectl apply -f cluster/plex/namespace.yaml
kubectl apply -f cluster/plex/pvc.yaml
envsubst < cluster/plex/secret.yaml | kubectl apply -f -
kubectl apply -f cluster/plex/deployment.yaml
kubectl apply -f cluster/plex/service.yaml
```

### Deploy Without Plex Claim Token

If you don't want to use a claim token (you can link Plex manually later):

```bash
# Set empty token
export PLEX_CLAIM_TOKEN=""

# Then deploy normally
kubectl kustomize cluster/plex/ | envsubst | kubectl apply -f -
```

### Verify Deployment

```bash
# Check pod status
kubectl -n media get pods

# Check pod logs
kubectl -n media logs -f deployment/plex

# Check service
kubectl -n media get svc

# Describe pod for troubleshooting
kubectl -n media describe pod -l app=plex
```

### Access Plex

1. Find your Raspberry Pi's IP: `hostname -I`
2. Open in browser: `http://<raspberry-pi-ip>:32400/web`
3. Complete Plex setup wizard

---

## Troubleshooting

### Pod Won't Start

```bash
# Check pod events
kubectl -n media describe pod -l app=plex

# Common issues:
# - "no persistent volumes available" → Check PV paths exist
# - "ImagePullBackOff" → Check internet connection
# - "CrashLoopBackOff" → Check logs: kubectl -n media logs -l app=plex
```

### Storage Issues

```bash
# Verify PV and PVC are bound
kubectl get pv
kubectl -n media get pvc

# Check if paths exist on host
ls -la /mnt/external-hdd/plex/config
ls -la /mnt/external-hdd/media
```

### Network Issues

```bash
# Check if Plex is listening
sudo netstat -tlnp | grep 32400

# Check service
kubectl -n media get svc plex

# Test from another device
curl http://<pi-ip>:32400/identity
```

### Resource Issues

```bash
# Check node resources
kubectl top nodes
kubectl top pods -n media

# If Pi is running out of memory, reduce limits in deployment.yaml
```

---

## Learning Resources

### Kubernetes

- [Kubernetes Official Docs](https://kubernetes.io/docs/home/)
- [Kubernetes The Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way)
- [K3s Documentation](https://docs.k3s.io/)

### Plex

- [Plex Support](https://support.plex.tv/)
- [LinuxServer.io Plex Image](https://docs.linuxserver.io/images/docker-plex)

### Related Projects to Add

Once comfortable, consider adding:

- **Sonarr** - TV show management
- **Radarr** - Movie management
- **Prowlarr** - Indexer management
- **Overseerr** - Request management
- **Tautulli** - Plex statistics

---

## License

This project is for personal/educational use. Plex is a trademark of Plex, Inc.
