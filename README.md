# Setting Up a Kubernetes Cluster with Kind and Podman on macOS

A complete guide for getting a local Kubernetes cluster running using Kind and Podman on your Mac, and deploying an SSH-enabled RHEL pod for development and testing - no Docker required!

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

**IMPORTANT:** Kind defaults to Docker, so you need to tell it to use Podman instead.

Add this to your shell profile so you don't have to type it every time:

```bash
# For zsh (default on modern macOS)
echo 'export KIND_EXPERIMENTAL_PROVIDER=podman' >> ~/.zshrc
source ~/.zshrc

# For bash
echo 'export KIND_EXPERIMENTAL_PROVIDER=podman' >> ~/.bashrc
source ~/.bashrc
```

To verify it's set:
```bash
echo $KIND_EXPERIMENTAL_PROVIDER
# Should output: podman
```

### 3. Create Your Kubernetes Cluster

Now for the magic:

```bash
# Create a cluster using Podman (note: use hyphens, not underscores in cluster names!)
kind create cluster --name my-rhel-cluster
```

This will take a minute or two. You'll see output showing Kind pulling the necessary images and setting up your cluster.

**Note:** Kind's Podman support is experimental but generally stable. If you encounter issues, Docker is the more battle-tested option.

### 4. Verify Your Cluster

Check that everything is working:

```bash
# See your cluster
kind get clusters

# Check cluster info
kubectl cluster-info --context kind-my-rhel-cluster

# List all pods (you'll see system pods in kube-system namespace)
kubectl get pods --all-namespaces
```

Congratulations! 🎉 You now have a working Kubernetes cluster.

## Deploying an SSH-Enabled RHEL Pod

### Understanding the RHEL SSH Pod

The `rhel-ssh-pod.yaml` file creates a pod that:
- Runs the latest Red Hat Universal Base Image (RHEL 9)
- Installs OpenSSH server for remote access
- Installs Python 3 (useful for Ansible and automation)
- Installs Screen (for terminal session management)
- Configures SSH to allow root login
- Sets the root password to `Asdf_123`
- Starts the SSH daemon

This pod mimics a production RHEL server environment, allowing you to develop and test automation scripts (Ansible, Python) locally before deploying to production.

### The RHEL SSH Pod YAML

Create a file named `rhel-ssh-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: rhel-ssh-pod
  labels:
    app: rhel-ssh
spec:
  containers:
  - name: rhel
    image: registry.access.redhat.com/ubi9/ubi:latest
    command: ["/bin/bash"]
    args:
      - -c
      - |
        dnf install -y openssh-server python3 screen && \
        ssh-keygen -A && \
        sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config && \
        echo "root:Asdf_123" | chpasswd && \
        /usr/sbin/sshd -D
    ports:
    - containerPort: 22
```

### Deploy the Pod

```bash
# Apply the configuration
kubectl apply -f rhel-ssh-pod.yaml

# Check if it's running (may take 30-60 seconds to install packages)
kubectl get pods

# Watch the pod until it's Running
kubectl get pods -w
# Press Ctrl+C once STATUS shows "Running"

# View the installation logs
kubectl logs rhel-ssh-pod
```

You should see output showing the SSH server and other packages being installed.

### Access the Pod via SSH

Since Kind clusters run in containers, you need to forward the pod's SSH port to your Mac:

```bash
# Forward pod's port 22 to your Mac's port 2222
kubectl port-forward rhel-ssh-pod 2222:22
```

**Important:** Keep this terminal window open - the port forwarding stays active as long as the command is running. You'll see:
```
Forwarding from 127.0.0.1:2222 -> 22
Forwarding from [::1]:2222 -> 22
```

Now, in a **new terminal window**, SSH into the pod:

```bash
# SSH into the pod (password is Asdf_123)
ssh -p 2222 root@localhost

# You're now inside the RHEL container!
# Try some commands:
cat /etc/redhat-release
uname -a
whoami

# Exit when done
exit
```

### First Time SSH Connection

The first time you connect, SSH will ask you to verify the host key:

```
The authenticity of host '[localhost]:2222' can't be established.
ED25519 key fingerprint is SHA256:...
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

Type `yes` and press Enter.

### Dealing with Host Key Changes

If you delete and recreate the pod, you'll get a warning about the host key changing. Fix it with:

```bash
ssh-keygen -R "[localhost]:2222"
```

Or, to avoid this warning in development, add to `~/.ssh/config`:

```bash
cat >> ~/.ssh/config << 'EOF'

Host localhost
  StrictHostKeyChecking no
  UserKnownHostsFile=/dev/null
EOF
```

## Port Forwarding in Background

If you want port forwarding to run in the background:

```bash
# Run port-forward in background
kubectl port-forward rhel-ssh-pod 2222:22 &

# Check it's running
jobs

# Kill it later with
pkill -f "port-forward.*2222:22"
```

## Using the Pod for Automation Development

This setup is perfect for developing automation that will run on production RHEL servers:

### With Ansible

Create an inventory file `inventory.ini`:

```ini
[dev_servers]
rhel-ssh-pod ansible_host=localhost ansible_port=2222 ansible_user=root ansible_password=Asdf_123

[dev_servers:vars]
ansible_python_interpreter=/usr/bin/python3
```

Test your Ansible playbooks:

```bash
# Ping test
ansible dev_servers -i inventory.ini -m ping

# Run commands
ansible dev_servers -i inventory.ini -m command -a "ls -la"

# Run playbooks
ansible-playbook -i inventory.ini your-playbook.yml
```

### With Python (using Paramiko)

```python
import paramiko

ssh = paramiko.SSHClient()
ssh.set_missing_host_key_policy(paramiko.AutoAddPolicy())
ssh.connect('localhost', port=2222, username='root', password='Asdf_123')

# Execute commands
stdin, stdout, stderr = ssh.exec_command('ls -la')
print(stdout.read().decode())

ssh.close()
```

## Common Commands

### Managing Your Cluster

```bash
# List all clusters
kind get clusters

# Delete your cluster
kind delete cluster --name my-rhel-cluster

# Create a cluster with custom configuration
kind create cluster --name my-rhel-cluster --config kind-config.yaml
```

### Managing Pods

```bash
# List all pods
kubectl get pods

# Get detailed pod info
kubectl describe pod rhel-ssh-pod

# View pod logs
kubectl logs rhel-ssh-pod

# Follow logs in real-time
kubectl logs -f rhel-ssh-pod

# Delete the pod
kubectl delete pod rhel-ssh-pod

# Recreate the pod
kubectl apply -f rhel-ssh-pod.yaml

# Execute commands directly (without SSH)
kubectl exec rhel-ssh-pod -- ls -la
kubectl exec -it rhel-ssh-pod -- /bin/bash
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

### Port Forwarding Management

```bash
# Forward in foreground (keeps terminal open)
kubectl port-forward rhel-ssh-pod 2222:22

# Forward in background
kubectl port-forward rhel-ssh-pod 2222:22 &

# Check background jobs
jobs

# Kill specific port forward
pkill -f "port-forward.*2222:22"

# Forward multiple ports
kubectl port-forward rhel-ssh-pod 2222:22 8080:80
```

## Troubleshooting

### "Cannot connect to the Docker daemon" Error

If you see this error when running Kind commands, you forgot to set `KIND_EXPERIMENTAL_PROVIDER=podman`:

```bash
# Add to your shell profile
echo 'export KIND_EXPERIMENTAL_PROVIDER=podman' >> ~/.zshrc
source ~/.zshrc

# Or prefix each command
KIND_EXPERIMENTAL_PROVIDER=podman kind get clusters
```

### "Cannot connect to Podman" Error

Your Podman machine isn't running:

```bash
podman machine start
```

### Pod Shows "Completed" or "Error" Status

Check the logs to see what happened:

```bash
kubectl logs rhel-ssh-pod
kubectl describe pod rhel-ssh-pod
```

If the pod completed, it means the SSH server didn't start properly. Delete and recreate:

```bash
kubectl delete pod rhel-ssh-pod
kubectl apply -f rhel-ssh-pod.yaml
```

### SSH Connection Refused

Make sure port forwarding is active:

```bash
# Check if port-forward is running
ps aux | grep "port-forward"

# Start it if not running
kubectl port-forward rhel-ssh-pod 2222:22
```

Also verify the pod is running:

```bash
kubectl get pods
# STATUS should be "Running"
```

### Invalid Cluster Name

Kubernetes names must match `^[a-z0-9.-]+$` - use hyphens, not underscores:

✅ `my-rhel-cluster`
❌ `my_rhel_cluster`

### Want to Use Docker Instead?

If experimental Podman support gives you trouble:

1. Start Docker Desktop
2. Remove `KIND_EXPERIMENTAL_PROVIDER=podman` from your shell profile
3. Run Kind commands normally

## Quick Reference

### Complete Workflow

```bash
# 1. Start Podman machine (if not running)
podman machine start

# 2. Create cluster (one time)
kind create cluster --name my-rhel-cluster

# 3. Deploy RHEL pod
kubectl apply -f rhel-ssh-pod.yaml

# 4. Wait for it to be ready
kubectl wait --for=condition=ready pod/rhel-ssh-pod --timeout=120s

# 5. Forward SSH port
kubectl port-forward rhel-ssh-pod 2222:22 &

# 6. SSH in
ssh -p 2222 root@localhost
# Password: Asdf_123
```

### Automated Deployment Script

Create `deploy-rhel.sh`:

```bash
#!/bin/bash

echo "Deploying RHEL SSH pod..."
kubectl apply -f rhel-ssh-pod.yaml

echo "Waiting for pod to be ready..."
kubectl wait --for=condition=ready pod/rhel-ssh-pod --timeout=120s

# Kill existing port-forward
pkill -f "port-forward.*2222:22" 2>/dev/null

# Start new port-forward in background
kubectl port-forward rhel-ssh-pod 2222:22 > /dev/null 2>&1 &

echo ""
echo "✅ Pod is ready and port forwarding is active!"
echo "SSH with: ssh -p 2222 root@localhost"
echo "Password: Asdf_123"
```

Make it executable:

```bash
chmod +x deploy-rhel.sh
./deploy-rhel.sh
```

## Cleanup

When you're done:

```bash
# Stop port forwarding
pkill -f "port-forward.*2222:22"

# Delete the pod
kubectl delete pod rhel-ssh-pod

# Delete the cluster
kind delete cluster --name my-rhel-cluster

# Stop Podman machine to free resources
podman machine stop
```

## What's Next?

Now that you have a development environment that mimics production RHEL servers, you can:

- Develop and test Ansible playbooks locally
- Write Python automation scripts with SSH/Paramiko
- Practice Linux administration commands
- Test configuration management
- Develop CI/CD scripts
- Experiment with containerized workflows
- Build and test applications in a RHEL environment

The beauty of this setup is that your automation scripts will work identically in production - just change the hostname/IP from `localhost:2222` to your production servers!

---

**Happy automating!** 🚀

**Resources:**
- Kind documentation: https://kind.sigs.k8s.io/
- Kubernetes documentation: https://kubernetes.io/docs/home/
- Podman documentation: https://podman.io/
- Red Hat UBI images: https://catalog.redhat.com/software/containers/ubi9/ubi/615bcf606feffc5384e8452e
- Ansible documentation: https://docs.ansible.com/
- Paramiko documentation: https://www.paramiko.org/
