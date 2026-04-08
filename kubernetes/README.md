# Kubernetes Crash Course

A complete beginner-to-intermediate guide covering what Kubernetes is, its core concepts, local setup using Minikube and kubectl, and how to deploy a real app with Deployment, Service, and HPA.

---

## Table of Contents

1. [What is Kubernetes?](#what-is-kubernetes)
2. [Core Concepts / Modules](#core-concepts--modules)
3. [Local Setup — Minikube + kubectl](#local-setup--minikube--kubectl)
   - [macOS](#macos)
   - [Windows](#windows)
   - [Linux](#linux)
4. [Start Minikube](#start-minikube)
5. [Run an App on Kubernetes Locally](#run-an-app-on-kubernetes-locally)
6. [deployment.yaml — explained](#deploymentyaml--explained)
7. [service.yaml — explained](#serviceyaml--explained)
8. [hpa.yaml — explained](#hpayaml--explained)
9. [Useful Commands Cheatsheet](#useful-commands-cheatsheet)

---

## What is Kubernetes?

Kubernetes (K8s) is an open-source **container orchestration platform** originally developed by Google and now maintained by the CNCF (Cloud Native Computing Foundation).

It automates:
- **Deployment** of containerized apps
- **Scaling** (up/down based on load)
- **Self-healing** (restarts failed containers automatically)
- **Load balancing** across multiple replicas
- **Rolling updates** without downtime

> **Real-life analogy:** Imagine you run a food delivery app like Zomato. On a normal day, 10 servers are enough. But on New Year's Eve, traffic spikes 10x. Without Kubernetes, your team manually spins up servers — and forgets to shut them down (hello, cloud bill 💸). With Kubernetes, it detects the load surge, auto-scales from 10 → 80 pods in minutes, and scales back down when traffic drops. Zero manual work.

> Think of Kubernetes as the **manager of a restaurant kitchen**. Docker is the individual cook (runs one dish/container). Kubernetes manages the entire kitchen — decides how many cooks are needed, replaces a cook who falls sick, and routes orders to the right station.

---

## Core Concepts / Modules

### Pod
The **smallest deployable unit** in Kubernetes. A Pod wraps one or more containers that share the same network namespace and storage.

```
Pod → runs your container(s)
```

> **Real-life example:** Think of a Pod as a single food stall at a food court. It serves one type of food (one app). The food court (cluster) can have many such stalls (pods).

If your Node.js API container crashes, Kubernetes detects it and **automatically restarts the Pod** — you don't get paged at 3 AM for it.

---

### Node
A physical or virtual **machine** in the cluster. Every Pod runs on a Node. Nodes are managed by the control plane.

> **Real-life example:** A Node is like a physical kitchen station (oven, grill, prep counter). The head chef (control plane) decides which dish (Pod) goes to which station (Node) based on available capacity.

---

### Cluster
A **group of Nodes** managed together by Kubernetes. One cluster = one control plane + many worker nodes.

> **Real-life example:** Your entire AWS/GCP infrastructure running one product is a cluster. Dev, staging, and prod can each be a separate cluster, or separate namespaces inside the same cluster.

---

### Deployment
A higher-level resource that manages a **ReplicaSet** (a group of identical Pods). You declare your desired state — Kubernetes makes it happen and keeps it that way.

```
Deployment → manages → ReplicaSet → manages → Pods
```

> **Real-life example:** You tell Kubernetes *"I always want 3 copies of my joke-api running."* You push a bad commit and one Pod crashes. Kubernetes immediately spins up a new one. You have 3 running again within seconds — without any manual intervention.

> You also use Deployments for **zero-downtime releases**: push a new Docker image → Kubernetes rolls out new Pods one by one while the old ones are still serving traffic → no users notice anything.

---

### Service
An abstraction that exposes a set of Pods as a **stable network endpoint**. Pods come and go (restarts, scaling), but the Service gives them a fixed IP/DNS name.

Types:
| Type | When to use |
|---|---|
| `ClusterIP` | Default. Internal communication between microservices inside the cluster |
| `NodePort` | Expose app outside the cluster for local dev / Minikube testing |
| `LoadBalancer` | Production — provision a cloud load balancer (AWS ELB, GCP LB) |
| `ExternalName` | Route traffic to an external service (e.g. a managed RDS database) |

> **Real-life example:** You have 5 replicas of your API Pod. Their IPs keep changing when they restart. A Service sits in front and gives callers one stable address: `http://joke-api-service`. The Service load-balances across all 5 Pods — you never hard-code a Pod IP.

---

### ConfigMap & Secret

- `ConfigMap` — stores **non-sensitive** config (env vars, config files, feature flags)
- `Secret` — stores **sensitive data** (passwords, API keys, tokens) — base64-encoded at rest

> **Real-life example:**
> - `ConfigMap`: Your app reads `NODE_ENV=production` or `LOG_LEVEL=debug` from a ConfigMap. No need to rebuild the Docker image to change a config value.
> - `Secret`: Your DB password is stored in a Kubernetes Secret, not hardcoded in your repo. The Pod reads it as an env var at runtime. If the password rotates, you update the Secret — not the codebase.

---

### Namespace
A way to **logically isolate** resources inside the same physical cluster.

> **Real-life example:** One Kubernetes cluster, three namespaces:
> - `dev` — developers test features here, resources are minimal
> - `staging` — QA team runs full tests, mirrors production config
> - `production` — real users, strict resource limits, monitored 24/7
>
> Each namespace has its own Pods, Services, and Secrets — completely isolated from the others.

---

### HPA (Horizontal Pod Autoscaler)
Automatically **scales the number of Pod replicas** up or down based on CPU, memory, or custom metrics.

> **Real-life example:** Your joke-api normally runs 2 Pods. A viral tweet sends 50x your normal traffic. HPA detects CPU usage hit 80% → scales to 10 Pods automatically. After the traffic dies down, it scales back to 2. You pay for compute only when you actually need it.

---

### Ingress
An API object that manages **external HTTP/HTTPS routing** to Services inside the cluster — like a smart reverse proxy.

> **Real-life example:** You have 3 microservices behind one domain:
> - `api.myapp.com/users` → routes to `user-service`
> - `api.myapp.com/orders` → routes to `order-service`
> - `api.myapp.com/jokes` → routes to `joke-api-service`
>
> Instead of 3 separate LoadBalancers (expensive!), one Ingress handles all routing rules. It also terminates TLS (HTTPS) in one place.

---

## Local Setup — Minikube + kubectl

> **Prerequisites (all platforms):** Docker Desktop must be installed and running before starting Minikube.

---

### macOS

#### Install kubectl

```bash
# Option 1 — Homebrew (recommended, easiest)
brew install kubectl

# Option 2 — Apple Silicon (M1/M2/M3) binary
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/darwin/arm64/kubectl"
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl

# Option 3 — Intel Mac binary
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/darwin/amd64/kubectl"
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl

# Verify
kubectl version --client
```

#### Install Minikube

```bash
# Option 1 — Homebrew (recommended)
brew install minikube

# Option 2 — Apple Silicon binary
curl -LO https://github.com/kubernetes/minikube/releases/latest/download/minikube-darwin-arm64
sudo install minikube-darwin-arm64 /usr/local/bin/minikube

# Option 3 — Intel Mac binary
curl -LO https://github.com/kubernetes/minikube/releases/latest/download/minikube-darwin-amd64
sudo install minikube-darwin-amd64 /usr/local/bin/minikube

# Verify
minikube version
```

---

### Windows

> Run all commands in **PowerShell as Administrator**.

#### Install kubectl

```powershell
# Option 1 — winget (recommended, built into Windows 10/11)
winget install -e --id Kubernetes.kubectl

# Option 2 — Chocolatey
choco install kubernetes-cli

# Option 3 — Scoop
scoop install kubectl

# Option 4 — Direct binary download (amd64)
curl.exe -LO "https://dl.k8s.io/release/v1.35.0/bin/windows/amd64/kubectl.exe"
# Move kubectl.exe to a folder that's in your PATH, e.g. C:\Windows\System32
# or add its folder to the PATH environment variable

# Create kubeconfig folder (needed if it doesn't exist)
cd ~
mkdir .kube
New-Item config -type file

# Verify
kubectl version --client
```

#### Install Minikube

```powershell
# Option 1 — winget (recommended)
winget install Kubernetes.minikube

# Option 2 — Chocolatey
choco install minikube

# Option 3 — Manual binary download + PATH setup
New-Item -Path 'c:\' -Name 'minikube' -ItemType Directory -Force
$ProgressPreference = 'SilentlyContinue'
Invoke-WebRequest -OutFile 'c:\minikube\minikube.exe' `
  -Uri 'https://github.com/kubernetes/minikube/releases/latest/download/minikube-windows-amd64.exe' `
  -UseBasicParsing

# Add minikube to system PATH
$oldPath = [Environment]::GetEnvironmentVariable('Path', [EnvironmentVariableTarget]::Machine)
if ($oldPath.Split(';') -inotcontains 'C:\minikube') {
  [Environment]::SetEnvironmentVariable('Path', $('{0};C:\minikube' -f $oldPath), [EnvironmentVariableTarget]::Machine)
}

# Verify (open a new PowerShell window after PATH update)
minikube version
```

---

### Linux

#### Install kubectl

```bash
# --- Option 1: Binary (works on any distro, x86-64) ---
# Downloads the latest stable version automatically
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

# ARM64 (e.g. Raspberry Pi, AWS Graviton)
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/arm64/kubectl"

# Install system-wide (requires sudo)
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Install without root (user-local)
chmod +x kubectl
mkdir -p ~/.local/bin
mv ./kubectl ~/.local/bin/kubectl
# Make sure ~/.local/bin is in your PATH (add to ~/.bashrc or ~/.zshrc):
# export PATH="$HOME/.local/bin:$PATH"


# --- Option 2: Debian / Ubuntu (apt) ---
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gnupg

# Add the Kubernetes signing key
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.35/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
sudo chmod 644 /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# Add the Kubernetes apt repository
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.35/deb/ /' | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo chmod 644 /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubectl


# --- Option 3: Red Hat / Fedora / CentOS (yum/dnf) ---
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.35/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.35/rpm/repodata/repomd.xml.key
EOF

sudo yum install -y kubectl


# Verify (all distros)
kubectl version --client
```

#### Install Minikube

```bash
# --- Option 1: Binary (recommended, x86-64) ---
curl -LO https://github.com/kubernetes/minikube/releases/latest/download/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube && rm minikube-linux-amd64

# --- Option 2: Debian package (.deb) for Ubuntu/Debian ---
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube_latest_amd64.deb
sudo dpkg -i minikube_latest_amd64.deb

# --- Option 3: RPM package for Fedora/CentOS/RHEL ---
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-latest.x86_64.rpm
sudo rpm -Uvh minikube-latest.x86_64.rpm

# Verify
minikube version
```

---

## Start Minikube

```bash
# Start a local single-node Kubernetes cluster using Docker as the driver
# Works the same on macOS, Windows, and Linux
minikube start --driver=docker

# Start with more resources — recommended when running real apps
# (default is 2 CPUs, 2GB RAM — often not enough)
minikube start --driver=docker --cpus=2 --memory=4096

# Check cluster status
minikube status

# Enable metrics-server — REQUIRED for HPA (autoscaling) to work
minikube addons enable metrics-server

# Enable ingress controller — needed for routing HTTP via Ingress rules
minikube addons enable ingress

# List all available addons and their current status
minikube addons list
```

---

## Run an App on Kubernetes Locally

After Minikube is running, apply the yaml files from the `yaml/` directory:

```bash
# Apply all manifests at once
kubectl apply -f yaml/

# Or apply individually (order matters: Deployment first, then Service, then HPA)
kubectl apply -f yaml/deployment.yaml
kubectl apply -f yaml/service.yaml
kubectl apply -f yaml/hpa.yaml

# Watch Pods come up in real time
kubectl get pods -w
```

### Access the app via Minikube

```bash
# NodePort service — Minikube gives you a URL to open in the browser
minikube service joke-api-service --url

# Or use port-forward to tunnel the service to localhost:8080
kubectl port-forward svc/joke-api-service 8080:80

# LoadBalancer type — run this in a separate terminal (keeps running in foreground)
minikube tunnel
```

---

## deployment.yaml — explained

See [`yaml/deployment.yaml`](yaml/deployment.yaml)

Key things configured and why:

| Field | What it does | Real-life example |
|---|---|---|
| `replicas: 2` | Always run 2 identical Pods | Like having 2 cashiers open at a store — if one gets sick, the other keeps serving |
| `selector.matchLabels` | Links this Deployment to its Pods | Like a manager knowing which employees are on their team by name tags |
| `image` | Docker image to run | The actual app binary — your Node.js API packed as a Docker image |
| `resources.requests` | Minimum CPU/memory guaranteed | Reserved seat on the Node — Kubernetes won't put this Pod somewhere it can't breathe |
| `resources.limits` | Maximum CPU/memory allowed | A circuit breaker — if the app leaks memory, it gets killed + restarted, not take down the whole Node |
| `livenessProbe` | Is the container alive? | Like a heartbeat monitor — if it flatlines, restart the patient (container) |
| `readinessProbe` | Is the container ready to serve traffic? | Like a chef signaling "dish is ready" before waiter picks it up — no half-cooked food goes to customers |
| `rollingUpdate` | Update strategy | Release v2 of your app one Pod at a time — users never see downtime |

---

## service.yaml — explained

See [`yaml/service.yaml`](yaml/service.yaml)

Key things configured and why:

| Field | What it does | Real-life example |
|---|---|---|
| `type: NodePort` | Exposes app outside the cluster | Like opening a door to the outside world — Minikube can now give you an accessible URL |
| `selector: app: joke-api` | Routes traffic to matching Pods | Like a receptionist directing all "joke" calls to the joke department, not accounting |
| `port: 80` | Port the Service listens on | What the outside world calls — standard HTTP port |
| `targetPort: 3000` | Port on the Pod that receives traffic | What your Node.js app actually listens on inside the container |
| `nodePort: 30080` | Fixed port on the Node's IP | The exact door number — `minikube-ip:30080` hits your app directly |

---

## hpa.yaml — explained

See [`yaml/hpa.yaml`](yaml/hpa.yaml)

Key things configured and why:

| Field | What it does | Real-life example |
|---|---|---|
| `scaleTargetRef` | Which Deployment to scale | Points the autoscaler at your joke-api — not some other service |
| `minReplicas: 2` | Never go below 2 Pods | Like always keeping at least 2 support agents on shift, even at 3 AM |
| `maxReplicas: 10` | Never exceed 10 Pods | A safety cap — prevents infinite scaling from a bug or DDoS from bankrupting you |
| `averageUtilization: 60` (CPU) | Scale up when CPU hits 60% | If your cooks are 60% busy, hire more before the queue builds up |
| `stabilizationWindowSeconds: 300` (scale down) | Wait 5 min before scaling down | Don't send cashiers home the moment the lunch rush drops — wait to confirm it's really over |

> **Requires** `metrics-server` addon: `minikube addons enable metrics-server`

---

## Useful Commands Cheatsheet

---

### Minikube Commands

```bash
# Start the cluster (uses Docker driver by default if Docker Desktop is running)
minikube start

# Start with specific resources — use this for running real apps locally
minikube start --driver=docker --cpus=2 --memory=4096

# Stop the cluster — preserves all your deployments/state for next time
minikube stop

# Delete the cluster entirely — useful for a clean slate when something is broken
minikube delete

# SSH into the Minikube node — useful when you want to inspect the node itself
# e.g. check Docker images available inside Minikube
minikube ssh

# Open the Kubernetes Web UI dashboard in your browser
# Great for visual exploration of pods, deployments, services
minikube dashboard

# Get the IP address of the Minikube node
# Use this to access NodePort services: http://<minikube-ip>:30080
minikube ip

# Get the full URL for a NodePort service (combines minikube ip + nodePort)
minikube service <service-name> --url

# List all addons and see which are enabled/disabled
minikube addons list

# Enable a specific addon
# Common ones: metrics-server, ingress, dashboard, registry
minikube addons enable <addon-name>

# Disable an addon
minikube addons disable <addon-name>

# ⭐ Point your LOCAL Docker CLI to Minikube's internal Docker daemon
# This lets you build images directly into Minikube — no need to push to Docker Hub
# Run this before `docker build` when developing locally
eval $(minikube docker-env)          # macOS / Linux
# On Windows PowerShell:
# & minikube -p minikube docker-env --shell powershell | Invoke-Expression

# Undo the above — reset your Docker CLI back to the local daemon
eval $(minikube docker-env --unset)  # macOS / Linux

# Check the version of Kubernetes running in Minikube
minikube kubectl -- version
```

---

### kubectl — Cluster Info

```bash
# Show the API server URL and cluster info
# Use to confirm kubectl is pointing at the right cluster
kubectl cluster-info

# Show all nodes in the cluster (in Minikube there's only one)
kubectl get nodes

# Detailed info about a specific node — shows capacity, conditions, running pods
kubectl describe node <node-name>

# Show which cluster/context kubectl is currently talking to
# Essential when working with multiple clusters (local, dev, prod)
kubectl config current-context

# List ALL contexts (clusters) kubectl knows about
kubectl config get-contexts

# Switch kubectl to point at a different cluster
# e.g. switch from minikube to your production EKS cluster
kubectl config use-context <context-name>
```

---

### kubectl — Namespaces

```bash
# List all namespaces in the cluster
kubectl get namespaces

# Create a new namespace (isolate dev/staging/prod environments)
kubectl create namespace <name>

# Run any kubectl command inside a specific namespace using -n
kubectl get pods -n <namespace>

# Set a default namespace so you don't have to type -n every time
kubectl config set-context --current --namespace=<name>

# Delete a namespace — WARNING: deletes everything inside it (pods, services, etc.)
kubectl delete namespace <name>
```

---

### kubectl — Pods

```bash
# List all pods in the current namespace
kubectl get pods

# List pods in a specific namespace
kubectl get pods -n <namespace>

# List pods with extra columns: which Node they're on, their internal IP
kubectl get pods -o wide

# Detailed info about a pod — great first step when a Pod is stuck or crashing
# Shows: Events, restart count, probe status, volumes, env vars
kubectl describe pod <pod-name>

# Stream live logs from a pod — like `tail -f` for your container
# Use when debugging a running app or watching startup errors
kubectl logs -f <pod-name>

# Get logs from a specific container inside a pod (for multi-container pods)
kubectl logs <pod-name> -c <container-name>

# Get logs from the previous container instance (if the pod crashed and restarted)
# Very useful for seeing WHY the pod crashed
kubectl logs <pod-name> --previous

# Open an interactive shell inside a running pod
# Like SSH-ing into your container — great for checking env vars, files, connectivity
kubectl exec -it <pod-name> -- /bin/sh

# Delete a pod — Kubernetes will automatically recreate it (because of the Deployment)
# Use to force a pod restart without changing anything
kubectl delete pod <pod-name>

# Watch pods update in real time — useful during deploys or scaling events
kubectl get pods -w
```

---

### kubectl — Deployments

```bash
# List all deployments and their desired/current/available replica counts
kubectl get deployments

# Detailed info — shows replica sets, strategy, conditions, events
kubectl describe deployment <deployment-name>

# Apply a deployment from a yaml file
# If the deployment exists, it updates it. If not, it creates it.
kubectl apply -f deployment.yaml

# Scale replicas manually — use for immediate one-off scaling
# (HPA will override this automatically if configured)
kubectl scale deployment <deployment-name> --replicas=3

# Update the container image — triggers a rolling update with zero downtime
# e.g. deploy v2 of your joke-api
kubectl set image deployment/joke-api joke-api=joke-api:v2

# Check if a rolling update finished successfully
kubectl rollout status deployment/<deployment-name>

# See the history of all rollouts (previous versions you can roll back to)
kubectl rollout history deployment/<deployment-name>

# Rollback to the previous version — use immediately when a bad deploy goes out
kubectl rollout undo deployment/<deployment-name>

# Rollback to a specific version from the history
kubectl rollout undo deployment/<deployment-name> --to-revision=2

# Delete a deployment (also removes its ReplicaSet and Pods)
kubectl delete deployment <deployment-name>
```

---

### kubectl — Services

```bash
# List all services and their type, cluster-IP, external-IP, ports
kubectl get services
kubectl get svc   # shorthand

# Detailed info — shows which Pods the service is routing to (Endpoints)
# Use when traffic isn't reaching your pods — check if endpoints are populated
kubectl describe svc <service-name>

# Quick way to create a ClusterIP service (internal only) without a yaml file
kubectl expose deployment <deployment-name> --port=80 --target-port=3000

# Quick NodePort service — accessible from outside the cluster
kubectl expose deployment <deployment-name> --type=NodePort --port=80 --target-port=3000

# Quick LoadBalancer service — use with `minikube tunnel` locally, or on AWS/GCP
kubectl expose deployment <deployment-name> --type=LoadBalancer --port=80 --target-port=3000

# Apply a service from a yaml file (preferred approach — version controlled)
kubectl apply -f service.yaml

# Delete a service
kubectl delete svc <service-name>

# Port-forward a service to your localhost — quickest way to test without NodePort
# Maps localhost:8080 → Service port 80 → Pod port 3000
# Great for local dev when you just want to hit your API fast
kubectl port-forward svc/<service-name> 8080:80
```

---

### kubectl — HPA (Horizontal Pod Autoscaler)

```bash
# List all HPAs — shows current vs desired replicas, and current CPU%
kubectl get hpa

# Detailed info — shows scaling events, thresholds, cooldown windows
kubectl describe hpa <hpa-name>

# Apply HPA from a yaml file (recommended — keeps config in version control)
kubectl apply -f hpa.yaml

# Create an HPA imperatively without yaml
# Scales between 2–10 replicas when CPU exceeds 50%
kubectl autoscale deployment <deployment-name> --cpu-percent=50 --min=2 --max=10

# Delete an HPA (stops autoscaling — replicas stay at current count)
kubectl delete hpa <hpa-name>
```

---

### kubectl — ConfigMaps & Secrets

```bash
# Create a ConfigMap from inline key-value pairs
# Use for app config like NODE_ENV, LOG_LEVEL, API_URL
kubectl create configmap <name> --from-literal=NODE_ENV=production --from-literal=LOG_LEVEL=info

# Create a ConfigMap from a file (e.g. an .env file or properties file)
kubectl create configmap <name> --from-file=config.properties

# List all ConfigMaps
kubectl get configmaps

# View the contents of a ConfigMap
kubectl describe configmap <name>


# Create a Secret from inline key-value pairs
# Values are base64-encoded automatically — never store raw secrets in code
kubectl create secret generic <name> --from-literal=DB_PASSWORD=mysecretpassword

# List all Secrets
kubectl get secrets

# Describe a Secret — shows keys but NOT values (for security)
kubectl describe secret <name>

# Decode a specific secret value (base64 decode)
kubectl get secret <name> -o jsonpath='{.data.DB_PASSWORD}' | base64 --decode
```

---

### kubectl — General / Debugging

```bash
# Get ALL resources in the current namespace at once
# Great for a quick overview of what's running
kubectl get all

# Delete everything defined in a yaml file (reverse of apply)
# Use for full teardown of an environment
kubectl delete -f yaml/

# Watch any resource type for live changes (pods, deployments, etc.)
kubectl get pods -w

# Export a resource as YAML — useful to see the full live config
# or to copy a resource definition as a starting point
kubectl get deployment <name> -o yaml

# Export as JSON (useful for scripting / jq processing)
kubectl get pod <name> -o json

# Show CPU and memory usage of Nodes (requires metrics-server)
kubectl top nodes

# Show CPU and memory usage of Pods
kubectl top pods

# Show recent cluster events sorted by time — BEST first stop for debugging
# e.g. why did a pod fail to schedule? Why did it get OOMKilled?
kubectl get events --sort-by='.lastTimestamp'

# Filter events for a specific resource
kubectl get events --field-selector involvedObject.name=<pod-name>

# Force delete a stuck pod (when normal delete hangs)
kubectl delete pod <pod-name> --force --grace-period=0
```

---

## Quick Workflow Summary

```bash
# 1. Start Minikube with enough resources
minikube start --driver=docker --cpus=2 --memory=4096

# 2. Enable metrics-server (needed for HPA autoscaling)
minikube addons enable metrics-server

# 3. Point Docker CLI at Minikube so you can build locally without a registry
eval $(minikube docker-env)

# 4. Build your Docker image directly into Minikube
docker build -t joke-api:latest .

# 5. Apply all Kubernetes manifests
kubectl apply -f yaml/

# 6. Watch Pods come up (Ctrl+C to exit watch)
kubectl get pods -w

# 7. Get the app URL and open it
minikube service joke-api-service --url

# 8. Check HPA is watching (CPU% and replica count)
kubectl get hpa

# 9. Clean up when done
kubectl delete -f yaml/
minikube stop
```
