# Kubernetes Installation Guide - CKA Exam Reference

**Complete guide from VM setup through cluster installation for CKA exam preparation.**

---

## Table of Contents
1. [VM Setup with Vagrant + KVM](#vm-setup)
2. [Initial System Configuration](#initial-config)
3. [Container Runtime Installation](#container-runtime)
4. [Kubernetes Components Installation](#k8s-components)
5. [Control Plane Initialization](#control-plane)
6. [CNI Network Plugin](#cni-plugin)
7. [Join Worker Nodes](#worker-nodes)
8. [Cluster Verification](#verification)
9. [Troubleshooting](#troubleshooting)

---

## Quick Reference Links

**Official Kubernetes Documentation Paths:**
- **kubeadm installation**: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/
- **Creating cluster with kubeadm**: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/
- **kubeadm init reference**: https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-init/
- **kubeadm join reference**: https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-join/
- **Container runtimes**: https://kubernetes.io/docs/setup/production-environment/container-runtimes/
- **Network addons**: https://kubernetes.io/docs/concepts/cluster-administration/addons/

- **Container runtimes**: https://kubernetes.io/docs/setup/production-environment/container-runtimes/
- **Network addons**: https://kubernetes.io/docs/concepts/cluster-administration/addons/

---

## <a name="vm-setup"></a>Phase 0: VM Setup with Vagrant + KVM

**Why VMs for CKA prep:** The exam uses VMs, not containers or physical hardware. Practicing on VMs gives you the most realistic experience for kubeadm bootstrap, node troubleshooting, and cluster operations.

### Install VirtualBox vs KVM

**Check if KVM is already loaded:**
```bash
lsmod | grep kvm
```

If KVM modules are loaded, **use KVM** (native Linux virtualization). If not loaded and you prefer VirtualBox, that works too.

**This guide assumes KVM/libvirt** (better performance on Linux).

### Install KVM and Libvirt

```bash
# Install KVM and management tools
sudo apt update
sudo apt install -y qemu-kvm libvirt-daemon-system libvirt-clients bridge-utils virtinst virt-manager

# Install dependencies for vagrant-libvirt
sudo apt install -y ruby-dev libvirt-dev build-essential libxml2-dev libxslt1-dev zlib1g-dev

# Add your user to required groups
sudo usermod -aG libvirt $USER
sudo usermod -aG kvm $USER

# Start and enable libvirt
sudo systemctl start libvirtd
sudo systemctl enable libvirtd

# Verify libvirt is running
sudo systemctl status libvirtd
```

**Important:** Log out and back in (or reboot) for group membership to take effect.

**Verify KVM is working:**
```bash
# Check groups
groups | grep libvirt

# Check KVM acceleration available
ls -l /dev/kvm

# Check virsh works
virsh list --all
```

### Install Vagrant

```bash
# Download and add HashiCorp GPG key
wget -O- https://apt.releases.hashicorp.com/gpg | \
  sudo gpg --dearmor --yes -o /etc/apt/keyrings/hashicorp-archive-keyring.gpg

# Add HashiCorp repository
echo "deb [signed-by=/etc/apt/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | \
  sudo tee /etc/apt/sources.list.d/hashicorp.list

# Install Vagrant
sudo apt update
sudo apt install -y vagrant

# Verify installation
vagrant --version
```

### Install vagrant-libvirt Plugin

```bash
# Install plugin (takes 5-10 minutes)
vagrant plugin install vagrant-libvirt

# Verify
vagrant plugin list
# Should show: vagrant-libvirt (...)
```

**If plugin installation fails:**
```bash
# Install additional dependencies
sudo apt install -y libguestfs-tools

# Try again
vagrant plugin install vagrant-libvirt
```

### Create Lab Directory and Vagrantfile

```bash
# Create directory
mkdir -p ~/k8s-lab
cd ~/k8s-lab

# Create Vagrantfile
vim Vagrantfile
```

**Vagrantfile contents:**
```ruby
# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  # Use generic Ubuntu box (works better with libvirt)
  config.vm.box = "generic/ubuntu2204"
  
  # Disable default synced folder (can cause issues)
  config.vm.synced_folder ".", "/vagrant", disabled: true

  # Control plane node
  config.vm.define "control" do |control|
    control.vm.hostname = "k8s-control"
    control.vm.network "private_network", ip: "192.168.121.10"
    
    control.vm.provider "libvirt" do |lv|
      lv.memory = "2048"
      lv.cpus = 2
    end
  end

  # Worker nodes
  (1..3).each do |i|
    config.vm.define "worker#{i}" do |worker|
      worker.vm.hostname = "k8s-worker#{i}"
      worker.vm.network "private_network", ip: "192.168.121.#{10+i}"
      
      worker.vm.provider "libvirt" do |lv|
        lv.memory = "1024"
        lv.cpus = 1
      end
    end
  end
end
```

**Understanding the Vagrantfile:**
- `config.vm.box = "generic/ubuntu2204"` - Ubuntu 22.04 LTS image
- `config.vm.synced_folder ... disabled: true` - Disables folder sharing (optional, prevents issues)
- `control.vm.network "private_network", ip: "192.168.121.10"` - Creates isolated network for cluster
- `lv.memory = "2048"` - 2GB RAM for control plane (minimum requirement)
- `lv.cpus = 2` - 2 CPU cores

**Network layout:**
- Control plane: 192.168.121.10
- Worker 1: 192.168.121.11
- Worker 2: 192.168.121.12
- Worker 3: 192.168.121.13

### Start VMs

```bash
cd ~/k8s-lab

# Start all VMs (first time downloads ~500MB Ubuntu image)
vagrant up

# This takes 5-10 minutes on first run
```

**Expected output:**
```
Bringing machine 'control' up with 'libvirt' provider...
Bringing machine 'worker1' up with 'libvirt' provider...
Bringing machine 'worker2' up with 'libvirt' provider...
Bringing machine 'worker3' up with 'libvirt' provider...
==> control: Creating image (snapshot of base box volume).
==> control: Creating domain with the following settings...
==> control: Starting domain.
...
```

**Note:** You may see warnings like `[fog][WARNING] Unrecognized arguments: libvirt_ip_command` - these are harmless.

### Verify VMs Are Running

```bash
# Check Vagrant status
vagrant status

# Should show all 4 VMs as "running (libvirt)"

# Check with virsh
virsh list --all

# Should show 4 running VMs

# Test SSH access
vagrant ssh control -c "hostname"
# Should output: k8s-control

vagrant ssh worker1 -c "hostname"
# Should output: k8s-worker1
```

### Common Vagrant Commands

```bash
# Start all VMs
vagrant up

# Start specific VM
vagrant up control

# SSH to VM
vagrant ssh control

# Check status
vagrant status

# Shutdown VMs
vagrant halt

# Destroy VMs (delete completely)
vagrant destroy -f

# Rebuild VMs with new Vagrantfile
vagrant reload

# Suspend VMs (save state)
vagrant suspend

# Resume suspended VMs
vagrant resume
```

### VM Network Troubleshooting

**If VMs can't reach each other:**
```bash
# Check libvirt default network
virsh net-list --all

# If 'default' network is inactive
virsh net-start default
virsh net-autostart default

# Try vagrant up again
```

**If vagrant up fails with network errors:**
```bash
# Restart libvirt
sudo systemctl restart libvirtd

# Check libvirt logs
sudo journalctl -u libvirtd -n 50
```

---

## <a name="initial-config"></a>Phase 1: Initial System Configuration

**Run on ALL nodes (control plane + all 3 workers)**

This section prepares each VM with the prerequisites for Kubernetes.

### Open Multiple Terminal Windows

For efficiency, open 4 terminal windows:

**Terminal 1 - Control Plane:**
```bash
cd ~/k8s-lab
vagrant ssh control
```

**Terminal 2 - Worker 1:**
```bash
cd ~/k8s-lab
vagrant ssh worker1
```

**Terminal 3 - Worker 2:**
```bash
cd ~/k8s-lab
vagrant ssh worker2
```

**Terminal 4 - Worker 3:**
```bash
cd ~/k8s-lab
vagrant ssh worker3
```

### Update System

**Run on each VM:**
```bash
# Update package lists and upgrade packages
sudo apt update && sudo apt upgrade -y

# This may take 5-10 minutes
# May show needrestart dialog - select "Yes" to restart services
```

**About needrestart dialog:**
- Appears when services need restart after library updates
- Select "Yes" or press Enter to continue
- To avoid prompt: `sudo DEBIAN_FRONTEND=noninteractive apt upgrade -y`

### Configure /etc/hosts

**Run on each VM:**

**Option 1 - Fast (heredoc):**
```bash
cat <<EOF | sudo tee -a /etc/hosts
192.168.121.10 k8s-control
192.168.121.11 k8s-worker1
192.168.121.12 k8s-worker2
192.168.121.13 k8s-worker3
EOF
```

**Option 2 - Manual (for understanding):**
```bash
# View current file
cat /etc/hosts

# Edit file
sudo vim /etc/hosts

# Press G (go to end)
# Press o (new line below, insert mode)
# Type these lines:
192.168.121.10 k8s-control
192.168.121.11 k8s-worker1
192.168.121.12 k8s-worker2
192.168.121.13 k8s-worker3

# Press ESC
# Type :wq
```

**Why this matters:** Allows nodes to find each other by hostname instead of IP. Makes Kubernetes configuration cleaner and more readable.

**Verify hostname resolution:**
```bash
ping -c 2 k8s-control
ping -c 2 k8s-worker1
# Should get responses
```

### Disable Swap

**Run on each VM:**
```bash
# Turn off swap immediately
sudo swapoff -a

# Disable swap permanently (comment out swap line in fstab)
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# Verify swap is disabled (Swap line should show 0)
free -h
```

**Why disable swap:** Kubernetes requires predictable memory management. Swap introduces performance variability. kubeadm will refuse to initialize if swap is enabled.

### Load Kernel Modules

**Run on each VM:**
```bash
# Load modules immediately
sudo modprobe overlay
sudo modprobe br_netfilter

# Make modules load on boot
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

# Verify modules are loaded
lsmod | grep overlay
lsmod | grep br_netfilter
# Should show both modules
```

**What these modules do:**
- **overlay**: Enables overlay filesystem (container image layers)
- **br_netfilter**: Allows iptables to see bridged traffic (pod networking)

### Configure Kernel Parameters

**Run on each VM:**
```bash
# Set kernel parameters
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply settings without reboot
sudo sysctl --system

# Verify settings
sudo sysctl net.bridge.bridge-nf-call-iptables net.ipv4.ip_forward
# Both should return 1
```

**What these settings do:**
- **net.bridge.bridge-nf-call-iptables = 1**: Bridge traffic goes through iptables rules
- **net.ipv4.ip_forward = 1**: Enable packet forwarding (routing between networks)

**Why these matter:** Required for Kubernetes pod networking. Without them, pods can't communicate across nodes.

### Verification Checklist

**Run on each VM to verify prerequisites:**
```bash
echo "=== Hostname ==="
hostname

echo -e "\n=== IP Address ==="
ip addr show | grep "inet 192.168.121"

echo -e "\n=== Hostname Resolution ==="
ping -c 1 k8s-control > /dev/null && echo "✓ Can reach k8s-control" || echo "✗ Cannot reach k8s-control"

echo -e "\n=== Swap Status ==="
free -h | grep Swap

echo -e "\n=== Kernel Modules ==="
lsmod | grep overlay > /dev/null && echo "✓ overlay loaded" || echo "✗ overlay not loaded"
lsmod | grep br_netfilter > /dev/null && echo "✓ br_netfilter loaded" || echo "✗ br_netfilter not loaded"

echo -e "\n=== Sysctl Settings ==="
sudo sysctl net.bridge.bridge-nf-call-iptables net.ipv4.ip_forward
```

**All checks should pass before proceeding to container runtime installation.**

---

## Prerequisites Checklist

Before starting, verify on **ALL nodes**:

```bash
# Verify swap is disabled
free -h  # Swap line should show 0

# Verify kernel modules loaded
lsmod | grep overlay
lsmod | grep br_netfilter

# Verify sysctl settings
sudo sysctl net.bridge.bridge-nf-call-iptables
sudo sysctl net.ipv4.ip_forward
# Both should return 1

# Verify hostname resolution
ping -c 2 k8s-control
ping -c 2 k8s-worker1
```

---

## <a name="container-runtime"></a>Phase 2: Container Runtime Installation

**Run on ALL nodes (control plane + all workers)**

**Docs reference**: https://kubernetes.io/docs/setup/production-environment/container-runtimes/#containerd

### Install containerd

```bash
# Install containerd package
sudo apt install -y containerd

# Create configuration directory
sudo mkdir -p /etc/containerd

# Generate default configuration
containerd config default | sudo tee /etc/containerd/config.toml
```

### Configure SystemdCgroup (CRITICAL)

**Why this matters:** Kubernetes (kubelet) uses systemd for cgroup management. Containerd must use the same cgroup driver or pods will fail to start.

```bash
# Edit containerd config
sudo vim /etc/containerd/config.toml

# Find this line (around line 125):
#   SystemdCgroup = false
# Change to:
#   SystemdCgroup = true

# Or do it automatically:
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
```

### Start containerd

```bash
# Restart with new configuration
sudo systemctl restart containerd

# Enable to start on boot
sudo systemctl enable containerd

# Verify it's running
sudo systemctl status containerd

# Test containerd works
sudo ctr version
```

**Troubleshooting:**
```bash
# If containerd fails to start:
sudo journalctl -u containerd -n 50

# Common issues:
# - Syntax error in config.toml
# - Missing kernel modules (check overlay, br_netfilter)
```

---

## <a name="k8s-components"></a>Phase 3: Kubernetes Components Installation

**Run on ALL nodes (control plane + all workers)**

**Docs reference**: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/

### Add Kubernetes Repository

```bash
# Install prerequisites
sudo apt install -y apt-transport-https ca-certificates curl gpg

# Create keyrings directory
sudo mkdir -p /etc/apt/keyrings

# Download Kubernetes GPG key (modern method, no apt-key)
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# Add Kubernetes repository
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list

# Update package list
sudo apt update
```

**Version Note:** Using v1.28 here. For different versions, change the URL:
- v1.29: `https://pkgs.k8s.io/core:/stable:/v1.29/deb/`
- v1.30: `https://pkgs.k8s.io/core:/stable:/v1.30/deb/`

### Install kubeadm, kubelet, kubectl

```bash
# Install all three components
sudo apt install -y kubelet kubeadm kubectl

# Prevent automatic updates (controlled upgrades only)
sudo apt-mark hold kubelet kubeadm kubectl

# Verify versions
kubeadm version
kubelet --version
kubectl version --client
```

**What each component does:**
- **kubeadm**: Cluster bootstrap and management tool
- **kubelet**: Node agent that runs on every node
- **kubectl**: Command-line tool to interact with cluster

### Enable kubelet

```bash
# Enable kubelet service (will fail to start until cluster is initialized)
sudo systemctl enable kubelet

# Check status (it's normal for it to be in "activating" state now)
sudo systemctl status kubelet
```

**Why kubelet is failing:** It's waiting for cluster configuration from kubeadm init.

---

## <a name="control-plane"></a>Phase 4: Initialize Control Plane

**Run ONLY on the control plane node**

**Docs reference**: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/

### Understanding kubeadm init Parameters

```bash
sudo kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \
  --apiserver-advertise-address=192.168.121.10
```

**Parameter breakdown:**

**`--pod-network-cidr=10.244.0.0/16`**
- Defines IP range for pods
- Must not overlap with:
  - Your node network (192.168.121.0/24)
  - Service CIDR (default 10.96.0.0/12)
- Different CNI plugins expect different CIDRs:
  - Flannel: 10.244.0.0/16
  - Calico: 192.168.0.0/16
  - Weave: 10.32.0.0/12

**Understanding CIDR notation:**
```
10.244.0.0/16
│         │
│         └─ First 16 bits are network, rest for hosts
└─ Starting IP

/16 = 65,534 available IPs for pods
Each node gets a subset (usually /24 = 254 IPs)
```

**`--apiserver-advertise-address=192.168.121.10`**
- IP address where API server listens
- Must be reachable by all nodes
- Use the control plane's IP on the cluster network

**Find your control plane IP:**
```bash
ip addr show
# Or
hostname -I
```

### Run kubeadm init

```bash
# Initialize control plane (takes 2-5 minutes)
sudo kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \
  --apiserver-advertise-address=192.168.121.10
```

**What happens during init:**
1. Runs pre-flight checks (swap, ports, RAM, etc.)
2. Downloads container images
3. Generates certificates
4. Creates kubeconfig files
5. Starts control plane components (API server, etcd, scheduler, controller)
6. Installs CoreDNS and kube-proxy
7. Generates join token for workers

**Expected output includes:**
```
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.121.10:6443 --token abcdef.0123456789abcdef \
    --discovery-token-ca-cert-hash sha256:longhashhere...
```

**⚠️ SAVE THE JOIN COMMAND!** You'll need it for worker nodes.

### Configure kubectl Access

```bash
# Create kubectl config directory
mkdir -p $HOME/.kube

# Copy admin credentials
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config

# Fix ownership
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

**What this does:**
- Copies cluster admin credentials to your user
- kubectl reads from `~/.kube/config` by default
- This gives you full cluster access

### Verify Control Plane

```bash
# Check nodes (will show NotReady - networking not installed yet)
kubectl get nodes

# Expected output:
# NAME          STATUS     ROLES           AGE   VERSION
# k8s-control   NotReady   control-plane   30s   v1.28.x

# Check control plane pods
kubectl get pods -n kube-system

# Should see:
# - etcd
# - kube-apiserver
# - kube-controller-manager
# - kube-scheduler
# - kube-proxy
# - coredns (pending - waiting for networking)
```

**Important directories created:**
```bash
# Certificates
ls /etc/kubernetes/pki/

# Static pod manifests
ls /etc/kubernetes/manifests/

# Kubeconfig files
ls /etc/kubernetes/

# Kubelet config
cat /var/lib/kubelet/config.yaml
```

---

## <a name="cni-plugin"></a>Phase 5: Install CNI Network Plugin

**Run on control plane node**

**Docs reference**: https://kubernetes.io/docs/concepts/cluster-administration/addons/

Without a CNI plugin, pods cannot communicate. The control plane will stay "NotReady".

### Option A: Flannel (Simplest - Recommended for Learning)

**Flannel docs**: https://github.com/flannel-io/flannel

```bash
# Install Flannel
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml

# Watch pods start
kubectl get pods -n kube-system -w

# Press Ctrl+C when all flannel pods are Running
```

**What Flannel does:**
- Creates overlay network using VXLAN
- Assigns subnet to each node from pod CIDR
- Encapsulates pod traffic for cross-node communication

**Verify Flannel:**
```bash
# Check flannel daemonset
kubectl get daemonset -n kube-system

# Check node is now Ready
kubectl get nodes
# Should show: STATUS = Ready

# Check all system pods running
kubectl get pods -n kube-system
```

### Option B: Calico (Better for Production)

**Calico docs**: https://docs.tigera.io/calico/latest/getting-started/kubernetes/quickstart

```bash
# Install Calico operator
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/tigera-operator.yaml

# Install Calico custom resources
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/custom-resources.yaml

# Watch pods start
kubectl get pods -n calico-system -w
```

**Note:** Calico defaults to 192.168.0.0/16 for pod CIDR. If you used 10.244.0.0/16 with kubeadm init, you need to edit the custom resources:

```bash
# Download custom resources
curl -O https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/custom-resources.yaml

# Edit the file
vim custom-resources.yaml

# Find and change:
# cidr: 192.168.0.0/16
# To:
# cidr: 10.244.0.0/16

# Apply
kubectl apply -f custom-resources.yaml
```

---

## <a name="worker-nodes"></a>Phase 6: Join Worker Nodes

**Run on each worker node**

**Docs reference**: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#join-nodes

### Join Workers to Cluster

Use the join command from kubeadm init output:

```bash
# On each worker node:
sudo kubeadm join 192.168.121.10:6443 \
  --token abcdef.0123456789abcdef \
  --discovery-token-ca-cert-hash sha256:longhashhere...
```

**What happens during join:**
1. Connects to API server at specified IP:port
2. Validates using the token
3. Verifies API server certificate using the hash
4. Downloads cluster configuration
5. Starts kubelet with cluster config
6. Registers node with API server

**Expected output:**
```
[preflight] Running pre-flight checks
[preflight] Reading configuration from the cluster...
[preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Starting the kubelet
[kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...

This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
```

### If Join Token Expired

Tokens expire after 24 hours. Generate a new one:

```bash
# On control plane:
kubeadm token create --print-join-command

# This outputs a complete join command with fresh token
# Copy and run on worker nodes
```

### Verify Worker Nodes Joined

```bash
# On control plane:
kubectl get nodes

# Should see all nodes:
# NAME          STATUS   ROLES           AGE   VERSION
# k8s-control   Ready    control-plane   10m   v1.28.x
# k8s-worker1   Ready    <none>          2m    v1.28.x
# k8s-worker2   Ready    <none>          2m    v1.28.x
# k8s-worker3   Ready    <none>          2m    v1.28.x

# Check node details
kubectl get nodes -o wide

# Check all system pods across nodes
kubectl get pods -n kube-system -o wide
```

---

## <a name="verification"></a>Phase 7: Cluster Verification

**Run on control plane**

### Test Pod Creation

```bash
# Create test pod
kubectl run nginx --image=nginx

# Check pod status
kubectl get pods -o wide

# Should show pod running on one of the worker nodes

# Check pod IP (should be in pod CIDR range)
kubectl get pod nginx -o jsonpath='{.status.podIP}'
```

### Test Pod Networking

```bash
# Create a second test pod
kubectl run busybox --image=busybox -- sleep 3600

# Get nginx pod IP
NGINX_IP=$(kubectl get pod nginx -o jsonpath='{.status.podIP}')

# Test connectivity from busybox to nginx
kubectl exec busybox -- wget -O- $NGINX_IP

# Should return nginx welcome page HTML
```

### Test DNS Resolution

```bash
# Create a service for nginx
kubectl expose pod nginx --port=80

# Test DNS from busybox
kubectl exec busybox -- nslookup nginx

# Should resolve nginx service IP

# Test DNS for kubernetes service
kubectl exec busybox -- nslookup kubernetes.default

# Should resolve to 10.96.0.1 (API server service IP)
```

### Test Service Discovery

```bash
# Access nginx via service name
kubectl exec busybox -- wget -O- nginx

# Should return nginx welcome page

# Check service details
kubectl get svc nginx

# Should show ClusterIP (from service CIDR 10.96.0.0/12)
```

### Cleanup Test Resources

```bash
kubectl delete pod nginx busybox
kubectl delete svc nginx
```

---

## <a name="troubleshooting"></a>Common Issues and Troubleshooting

### Issue: Nodes Stay NotReady

**Diagnose:**
```bash
kubectl describe node <node-name>
# Look at Conditions section

kubectl get pods -n kube-system
# Check if CNI pods are running
```

**Common causes:**
- CNI not installed
- CNI pods not running
- Incorrect pod CIDR in kubeadm init

### Issue: Pods Stuck in Pending

**Diagnose:**
```bash
kubectl describe pod <pod-name>
# Look at Events section
```

**Common causes:**
- Not enough resources
- Node selector doesn't match
- Taints without tolerations

### Issue: Pods Can't Communicate

**Diagnose:**
```bash
# Check CNI pods
kubectl get pods -n kube-system -l app=flannel

# Check node routing
ip route

# Test direct pod IP connectivity
kubectl exec <pod> -- ping <other-pod-ip>
```

**Common causes:**
- CNI misconfigured
- Firewall blocking
- Wrong pod CIDR

### Issue: kubectl Commands Hang

**Diagnose:**
```bash
# Check API server is running
sudo crictl ps | grep kube-apiserver

# Check API server logs
sudo crictl logs <apiserver-container-id>

# Check kubelet status
sudo systemctl status kubelet
sudo journalctl -u kubelet -n 50
```

**Common causes:**
- API server crashed
- Certificate issues
- etcd not responding

### Issue: kubeadm join Fails

**Diagnose:**
```bash
# On worker, check kubelet logs
sudo journalctl -u kubelet -n 50

# Verify connectivity to control plane
telnet 192.168.121.10 6443

# Check token validity (on control plane)
kubeadm token list
```

**Common causes:**
- Token expired
- Network unreachable
- Firewall blocking port 6443

---

## Important File Locations

### Configuration Files

```bash
# Kubernetes configs
/etc/kubernetes/
  ├── admin.conf           # Admin kubeconfig
  ├── kubelet.conf         # Kubelet kubeconfig (auto-generated)
  ├── controller-manager.conf
  ├── scheduler.conf
  ├── manifests/           # Static pod manifests
  │   ├── etcd.yaml
  │   ├── kube-apiserver.yaml
  │   ├── kube-controller-manager.yaml
  │   └── kube-scheduler.yaml
  └── pki/                 # Certificates
      ├── ca.crt           # Cluster CA certificate
      ├── ca.key           # Cluster CA key
      └── ...

# Kubelet config
/var/lib/kubelet/
  ├── config.yaml          # Kubelet configuration
  └── kubeadm-flags.env    # Kubelet flags

# Container runtime
/etc/containerd/config.toml

# System configs
/etc/modules-load.d/k8s.conf
/etc/sysctl.d/k8s.conf
```

### Data Directories

```bash
# etcd data
/var/lib/etcd/

# Kubelet data
/var/lib/kubelet/

# Container images and data
/var/lib/containerd/
```

---

## Quick Command Reference

### Cluster Info

```bash
# Cluster info
kubectl cluster-info
kubectl get componentstatuses

# Node info
kubectl get nodes
kubectl describe node <node-name>

# System pods
kubectl get pods -n kube-system
kubectl get all -n kube-system
```

### Troubleshooting Commands

```bash
# Check kubelet
sudo systemctl status kubelet
sudo journalctl -u kubelet -f

# Check containerd
sudo systemctl status containerd
sudo journalctl -u containerd -f

# Check containers
sudo crictl ps
sudo crictl pods

# Check logs
sudo crictl logs <container-id>

# Check certificates
sudo kubeadm certs check-expiration

# Network debugging
kubectl run test --image=busybox -- sleep 3600
kubectl exec test -- nslookup kubernetes.default
kubectl exec test -- wget -O- <pod-ip>
```

### Useful kubectl Commands

```bash
# Get resources
kubectl get nodes/pods/svc/deploy -o wide
kubectl get all --all-namespaces

# Describe (detailed info)
kubectl describe node/pod/svc <name>

# Logs
kubectl logs <pod-name>
kubectl logs <pod-name> -c <container-name>
kubectl logs <pod-name> --previous

# Execute commands
kubectl exec <pod-name> -- <command>
kubectl exec -it <pod-name> -- /bin/bash

# Events
kubectl get events --sort-by=.metadata.creationTimestamp
kubectl get events -n kube-system
```

---

## Next Steps for CKA Prep

After your cluster is running:

1. **Practice cluster teardown and rebuild**
   - `vagrant destroy -f && vagrant up`
   - Run through setup again
   - Time yourself - aim for < 20 minutes

2. **Practice with different CNI plugins**
   - Rebuild with Calico instead of Flannel
   - Understand differences

3. **Practice troubleshooting scenarios**
   - Intentionally break components
   - Fix them without looking at notes

4. **Learn cluster operations**
   - Cluster upgrade (kubeadm upgrade)
   - etcd backup and restore
   - Certificate renewal
   - Node maintenance (drain, cordon)

5. **Practice application deployments**
   - Deployments, Services, ConfigMaps
   - Resource limits and quotas
   - Network policies
   - RBAC

**Remember:** The exam gives you access to kubernetes.io/docs. Practice navigating these documentation pages and using them efficiently.

---

## Downloading This Guide

To save this guide as a markdown file on your system:

**On your host machine (not in VMs):**
```bash
# Create the file
cat > ~/k8s-install-guide.md << 'ENDOFFILE'
```

Then copy the entire contents of this artifact and paste it, followed by:

```bash
ENDOFFILE
```

**Or download from the artifact viewer** using the download/copy button in the interface.

**To view offline:**
```bash
# View in terminal
less ~/k8s-install-guide.md

# Or open in your preferred markdown viewer
vim ~/k8s-install-guide.md
```