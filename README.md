## Welcome to your Kubernetes training. Building a cluster from scratch is a foundational exercise for any infrastructure engineer. It strips away the magic of managed services (like EKS or GKE) and shows you exactly how the control plane and worker nodes interact.

### For this environment, we will use a RHEL-based operating system (such as RHEL 9, Rocky Linux 9, or AlmaLinux 9). The commands below will utilize `dnf` and `systemd` tools.

### Let's begin.

## 1. Pre-requisites

**Target: ALL NODES** (Run these commands on Master and both Workers)

Before installing Kubernetes, we must prepare the operating system to host containers and handle Kubernetes networking safely.
---
**1.1 Set Hostnames**

Each node must have a unique hostname so the cluster can accurately identify and route traffic to it.
```bash
# On Master Node
sudo hostnamectl set-hostname k8s-master
```
```bash
# On Worker Node 1
sudo hostnamectl set-hostname k8s-worker1
```
```bash
# On Worker Node 2
sudo hostnamectl set-hostname k8s-worker2
```
---
**1.2 Configure Host Resolution**

Kubernetes nodes need to communicate by name. We will update the `/etc/hosts` file.
```bash
# Open the file on all nodes
sudo vi /etc/hosts
```
```bash
# Add the following lines (replace with your actual VM IPs)
192.168.x.x k8s-master
192.168.x.x k8s-worker1
192.168.x.x k8s-worker2
```

---
**1.3 Disable Swap**

**Why**: The kubelet (the Kubernetes node agent) is designed to manage resources like CPU and memory strictly. If swap is enabled, the OS can move memory pages to disk, completely bypassing the kubelet's resource limits and causing unpredictable performance.
```bash
# Disable swap immediately
sudo swapoff -a

# Make it permanent across reboots by commenting out the swap entry
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```
---

**1.4 Set SELinux to Permissive**

**Why**: Kubernetes components need to access the host filesystem and network in ways that strict SELinux policies often block by default. Setting it to permissive allows containers to function while still logging policy violations.
```bash
# Set SELinux to permissive immediately
sudo setenforce 0

# Make it permanent across reboots
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```
---
**1.5 Disable Firewall (Lab Environment)**

**Why**: Kubernetes requires various ports (6443, 10250, etc.) to be open. For a learning environment, disabling firewalld prevents network troubleshooting headaches. Note: In production, you would configure specific port rules instead.
```bash
sudo systemctl disable --now firewalld
``` 
---
**1.6 Load Kernel Modules**

**Why**: Container runtimes and network plugins require specific kernel modules. overlay is needed for the container filesystem, and br_netfilter allows iptables to see bridged traffic (crucial for pod-to-pod communication).
```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
```
---
**1.7 Configure Sysctl Parameters**

**Why**: We must instruct the Linux kernel to forward IPv4 traffic and let iptables process bridge traffic.
```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply the changes without rebooting
sudo sysctl --system
```
---
**1.8 Install and Configure containerd**

**Why**: Kubernetes removed Docker as a runtime (Dockershim) in v1.24. We will use containerd, the industry standard CRI (Container Runtime Interface).
```bash
# Add the Docker repository (which hosts containerd)
sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

# Install containerd
sudo dnf install -y containerd.io

# Generate the default configuration file
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null

# Configure the systemd cgroup driver
# Why: Both the OS and Kubernetes use systemd to manage services. 
# Using two different cgroup managers (like cgroupfs and systemd) causes instability.
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml

# Enable and start containerd
sudo systemctl enable --now containerd
```
---
## 2. Install Kubernetes Components
**Target: ALL NODES**

We need three main binaries:

- `kubeadm`: The command to bootstrap the cluster.

- `kubelet`: The component that runs on all machines and starts pods/containers.

- `kubectl`: The command-line utility to talk to the cluster.

**2.1 Add the Kubernetes Repository**
```bash
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.30/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.30/rpm/repodata/repomd.xml.key
EOF
```
---
**2.2 Install the Packages**
```bash
# Install the binaries and disable exclusions so they can be installed
sudo dnf install -y kubelet kubeadm kubectl --disableexcludes=kubernetes

# Enable the kubelet service (do NOT start it yet, kubeadm will do that)
sudo systemctl enable kubelet
```
---

## Master Node Setup
**Target: MASTER NODE ONLY**

We will now bootstrap the control plane. This process generates certificates, sets up the API server, etcd (the database), scheduler, and controller manager.

**3.1 Initialize the Cluster**
```bash
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
```
- `--pod-network-cidr`: This tells the cluster what block of IP addresses to assign to your Pods. We use `192.168.0.0/16` because it is the default requirement for the Calico network plugin we will install later.

**Expected Output:**
The screen will output a lot of text. At the very end, it will give you a `kubeadm join` command. Copy this command and save it in a notepad. You will need it for `Step 5`.

---
**3.2 Configure kubectl for your user**

Why: The kubectl command needs a configuration file (kubeconfig) to know where the API server is and how to authenticate.
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
---

**3.3 Verify Control Plane**
```bash
kubectl get nodes
```

- <mark>Expected result</mark>: The master node will show up, but its status will be `NotReady`. This is completely normal because we haven't installed the network plugin yet.
---

## 4. Install Pod Network (CNI)
**Target: MASTER NODE ONLY**

Why: Kubernetes does not handle networking natively. It requires a Container Network Interface (CNI) plugin to assign IPs to pods and route traffic between them across different nodes. We will use Calico, a highly robust, BGP-based network plugin.

**4.1 Deploy Calico**
```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.3/manifests/calico.yaml
```
---

**4.2 Verify Network Setup**
```bash
kubectl get pods -n kube-system
kubectl get nodes
```

- <mark>Expected result</mark>: You will see Calico pods spinning up. Within a few minutes, run `kubectl get nodes` again, and the master node status should change from `NotReady` to `Ready`.

---

## 5. Worker Node Setup
**Target: WORKER NODES ONLY**
We will now attach the worker nodes to the control plane.

**5.1 Join the Cluster**

Paste the kubeadm join command you saved from `Step 3.1` into both worker nodes. It will look something like this:
```bash
sudo kubeadm join <master-ip>:6443 --token <token> \
        --discovery-token-ca-cert-hash sha256:<hash>
```
- Token: Acts as a password so the master knows this node is authorized to join.

- Discovery Hash: Ensures the worker is talking to the genuine master node (preventing man-in-the-middle attacks).

Note: Tokens expire after 24 hours. If you are doing this days later, run `kubeadm token create --print-join-command` on the Master node to generate a fresh command.

---

## 6. Verification
**Target: MASTER NODE ONLY**

Let's ensure the whole cluster is healthy.
```bash
kubectl get nodes
```
- <mark>Expected Output</mark>: You should see your master and both workers listed, and all should have a status of `Ready`.

```bash
kubectl get pods -n kube-system -o wide
```

- <mark>Expected Output</mark>: You should see CoreDNS, etcd, API server, and Calico pods running and distributed across your nodes.

---

## 7. Test Deployment
**Target: MASTER NODE ONLY**

Let's prove the cluster actually works by deploying an Nginx web server.

**7.1 Create a Deployment**
```bash
kubectl create deployment nginx-test --image=nginx:latest
```
- **Why**: This tells Kubernetes to download the Nginx container image and run it as a Pod.

---

**7.2 Expose the Deployment**
```bash
kubectl expose deployment nginx-test --port=80 --type=NodePort
```
- **Why**: By default, Pod IPs are only reachable inside the cluster. NodePort opens a port (between 30000-32767) on the IP address of every VM in the cluster, routing traffic to the Nginx pod.

---

**7.3 Verify and Access**
```bash
kubectl get svc nginx-test
```
Look at the output under the `PORT(S)` column. It will look like `80:31456/TCP`. The number after the colon (`31456`) is your NodePort.

Now, curl that port using the IP of any of your worker VMs:
```bash
curl http://<worker1-ip>:31456
```