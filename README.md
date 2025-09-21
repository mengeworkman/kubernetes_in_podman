# Setting Up a Kubernetes Cluster with Kind and Podman on macOS

A complete guide for getting a local Kubernetes cluster running using Kind and Podman on your Mac, and deploying your first RHEL pod - no Docker required!

## Why Podman + Kind?

Podman is a Docker alternative that's daemon-less and rootless by default, making it more secure. Kind (Kubernetes IN Docker) is ironically named but works great with Podman too, giving you a real Kubernetes cluster perfect for local development and testing.

## Prerequisites

- macOS (Apple Silicon or Intel)
- Podman installed (`brew install podman`)
- Kind installed (`brew install kind`)
- kubectl installed (`brew install kubectl`)

## Step-by-Step Setup

### 1. Initialize Your Podman Machine

Podman on macOS needs a Linux VM to run containers. Let's set that up:

```bash
# Create a new Podman machine
podman machine init

# Start the machine
podman machine start

# Verify it's running
podman machine list
```

You should see output like:
```
NAME                    VM TYPE     CREATED         LAST UP            CPUS        MEMORY      DISK SIZE
podman-machine-default  applehv     1 minute ago    Currently running  5           2GiB        100GiB
```

**Pro tip:** The Podman machine will need to be running every time you want to use Kind. You can check its status anytime with `podman machine list`.

### 2. Configure Kind to Use Podman

**IMPORTANT:** Kind defaults to Docker, so you need to tell it to use Podman instead. You have two options:

#### Option A: Set it permanently (Recommended)

Add this to your shell profile so you don't have to type it every time:

```bash
# For zsh (default on modern macOS)
echo 'export KIND_EXPERIMENTAL_PROVIDER=podman' >> ~/.zshrc
source ~/.zshrc

# For bash
echo 'export KIND_EXPERIMENTAL_PROVIDER=podman' >> ~/.bashrc
source ~/.bashrc
```

#### Option B: Set it per command

Prefix every Kind command with `KIND_EXPERIMENTAL_PROVIDER=podman`:

```bash
KIND_EXPERIMENTAL_PROVIDER=podman kind create cluster --name my-cluster
KIND_EXPERIMENTAL_PROVIDER=podman kind get clusters
KIND_EXPERIMENTAL_PROVIDER=podman kind delete cluster --name my-cluster
```

**For the rest of this guide, we'll assume you've set it permanently (Option A).**

### 3. Create Your Kubernetes Cluster

Now for the magic:

```bash
# Create a cluster using Podman (note: use hyphens, not underscores in cluster names!)
kind create cluster --name my-cluster
```

This will take a minute or two. You'll see output showing Kind pulling the necessary images and setting up your cluster.

**Note:** Kind's Podman support is experimental but generally stable. If you encounter issues, Docker is the more battle-tested option.

### 4. Verify Your Cluster

Check that everything is working:

```bash
# See your cluster
kind get clusters

# Check cluster info
kubectl cluster-info --context kind-my-cluster

# List all pods (you'll see system pods in kube-system namespace)
kubectl get pods --all-namespaces
```

Congratulations! 🎉 You now have a working Kubernetes cluster.

### 5. Deploy Your First Pod (RHEL)

Let's deploy a RHEL pod to your cluster:

```bash
# Create a RHEL pod that stays running
kubectl run rhel-pod --image=registry.access.redhat.com/ubi9/ubi:latest --restart=Never -- /bin/bash -c "sleep infinity"

# Check if it's running
kubectl get pods
```

You should see:
```
NAME       READY   STATUS    RESTARTS   AGE
rhel-pod   1/1     Running   0          5s
```

**Note:** We're using Red Hat's Universal Base Image (UBI) which is freely available. The `sleep infinity` command keeps the container alive so you can interact with it.

Now access your RHEL pod:

```bash
# Get a shell inside the pod
kubectl exec -it rhel-pod -- /bin/bash

# You're now inside the RHEL container!
# Try some commands:
cat /etc/redhat-release
uname -a
whoami

# Exit when done
exit
```

## Alternative: Deploy RHEL with YAML

For a more production-like approach, create a file called `rhel-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: rhel-pod
  labels:
    app: rhel
spec:
  containers:
  - name: rhel-container
    image: registry.access.redhat.com/ubi9/ubi:latest
    command: ["/bin/bash", "-c", "sleep infinity"]
```

Then deploy it:

```bash
# Apply the configuration
kubectl apply -f rhel-pod.yaml

# Check status
kubectl get pods

# Access the pod
kubectl exec -it rhel-pod -- /bin/bash
```

## Common Commands

### Managing Your Cluster

```bash
# List all clusters
kind get clusters

# Delete your cluster
kind delete cluster --name my-cluster

# Create a cluster with custom configuration
kind create cluster --name my-cluster --config kind-config.yaml

# Get cluster details
kubectl cluster-info --context kind-my-cluster
```

### Managing Pods

```bash
# List all pods
kubectl get pods

# Get detailed pod info
kubectl describe pod rhel-pod

# View pod logs
kubectl logs rhel-pod

# Delete a pod
kubectl delete pod rhel-pod

# Get pods in all namespaces
kubectl get pods --all-namespaces
```

### Managing Podman Machine

```bash
# Stop the Podman machine
podman machine stop

# Start it again
podman machine start

# Check status
podman machine list

# SSH into the machine (for debugging)
podman machine ssh
```

### Working with kubectl

```bash
# Get nodes
kubectl get nodes

# Create a deployment (for automatic restarts and scaling)
kubectl create deployment nginx --image=nginx

# See your deployments
kubectl get deployments

# Scale a deployment
kubectl scale deployment nginx --replicas=3

# Clean up
kubectl delete deployment nginx
```

## Troubleshooting

### Pod Shows "Completed" Status

If your RHEL pod exits immediately with status `Completed`, it means the container didn't have a long-running process. Always use a command like `sleep infinity` to keep it alive:

```bash
kubectl delete pod rhel-pod
kubectl run rhel-pod --image=registry.access.redhat.com/ubi9/ubi:latest --restart=Never -- /bin/bash -c "sleep infinity"
```

### "Cannot connect to the Docker daemon" Error

If you see this error when running Kind commands:
```
Cannot connect to the Docker daemon at unix:///Users/.../.docker/run/docker.sock
```

You forgot to set `KIND_EXPERIMENTAL_PROVIDER=podman`! Either:

1. Add it to your shell profile (see Step 2, Option A above), or
2. Prefix your command with `KIND_EXPERIMENTAL_PROVIDER=podman`

### "Cannot connect to Podman" Error

If you see connection errors about Podman, your Podman machine isn't running:

```bash
podman machine start
```

### Invalid Cluster Name

Kubernetes names must match `^[a-z0-9.-]+$` - use hyphens, not underscores:

✅ `my-cluster`
❌ `my_cluster`

### Want to Use Docker Instead?

If experimental Podman support gives you trouble:

1. Start Docker Desktop
2. Remove the `KIND_EXPERIMENTAL_PROVIDER=podman` from your shell profile
3. Run Kind commands normally:
   ```bash
   kind create cluster --name my-cluster
   ```

## Quick Reference: Environment Variable

Remember, **all** Kind commands need to know you're using Podman:

```bash
# Without setting it permanently, you'd need:
KIND_EXPERIMENTAL_PROVIDER=podman kind create cluster --name my-cluster
KIND_EXPERIMENTAL_PROVIDER=podman kind get clusters
KIND_EXPERIMENTAL_PROVIDER=podman kind delete cluster --name my-cluster

# With it set permanently in your shell profile:
kind create cluster --name my-cluster
kind get clusters
kind delete cluster --name my-cluster
```

To check if it's set:
```bash
echo $KIND_EXPERIMENTAL_PROVIDER
# Should output: podman
```

## What's Next?

Now that you have a cluster with a running RHEL pod, try:

- Installing software in your RHEL pod (`yum install` or `dnf install`)
- Creating multiple pods with different images
- Exploring Kubernetes services and ingress
- Learning about persistent volumes
- Experimenting with Helm charts
- Creating deployments for automatic pod management
- Breaking things (it's local, go wild!)

## Cleanup

When you're done experimenting:

```bash
# Delete your pod
kubectl delete pod rhel-pod

# Delete your cluster
kind delete cluster --name my-cluster

# Stop Podman machine to free up resources
podman machine stop
```

---

**Happy clustering!** 🚀

Got questions or run into issues?
- Kind docs: https://kind.sigs.k8s.io/
- Kubernetes docs: https://kubernetes.io/docs/home/
- Red Hat UBI images: https://catalog.redhat.com/software/containers/ubi9/ubi/615bcf606feffc5384e8452e
