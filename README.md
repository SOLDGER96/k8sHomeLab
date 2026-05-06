
# Kubernetes Concepts Reference Guide
 
> A companion reference for the Kubernetes from-scratch installation guide. This page explains every major concept, component, and tool encountered during the cluster setup.
 
---
 
## Table of Contents
 
1. [Core Architecture](#1-core-architecture)
2. [Control Plane Components](#2-control-plane-components)
3. [Worker Node Components](#3-worker-node-components)
4. [Networking Concepts](#4-networking-concepts)
5. [Container Runtime](#5-container-runtime)
6. [Key Tools](#6-key-tools)
7. [Operating System Prerequisites](#7-operating-system-prerequisites)
8. [Kubernetes Objects](#8-kubernetes-objects)
---
 
## 1. Core Architecture
 
### What is Kubernetes?
Kubernetes (K8s) is an open-source **container orchestration platform**. It automates the deployment, scaling, and management of containerized applications across a cluster of machines.
 
### Cluster
A Kubernetes cluster is a set of machines (nodes) that work together to run containerized workloads. Every cluster has:
- **One or more Control Plane (Master) nodes** — the brain that makes decisions
- **One or more Worker nodes** — the muscle that runs the actual applications
```
┌─────────────────────────────────────────────────────────┐
│                     KUBERNETES CLUSTER                  │
│                                                         │
│   ┌──────────────────┐      ┌────────┐   ┌────────┐    │
│   │   Control Plane  │      │ Worker │   │ Worker │    │
│   │   (k8s-master)   │◄────►│  Node  │   │  Node  │    │
│   │                  │      │   1    │   │   2    │    │
│   └──────────────────┘      └────────┘   └────────┘    │
└─────────────────────────────────────────────────────────┘
```
 
### Node
A **node** is a single physical or virtual machine in the cluster. Each node runs a container runtime (containerd) and the kubelet agent, which communicates with the control plane.
 
---
 
## 2. Control Plane Components
 
The control plane manages the overall state of the cluster. All of the following run on the **master node**.
 
### API Server (`kube-apiserver`)
The **central hub** of Kubernetes. Every interaction with the cluster — whether from `kubectl`, a worker node, or an internal component — goes through the API server. It:
- Exposes the Kubernetes REST API on port `6443`
- Validates and processes all requests
- Acts as the gateway to the cluster's data store (etcd)
### etcd
A distributed **key-value store** that serves as Kubernetes' database. It stores the entire state of the cluster — every node, pod, config, and secret. If etcd is lost without a backup, the cluster's state is gone. In production, etcd is always run in a redundant, multi-node configuration.
 
### Scheduler (`kube-scheduler`)
The component responsible for **deciding which node a new Pod runs on**. It watches for unscheduled pods and assigns them to suitable nodes based on:
- Available CPU and memory
- Node taints and tolerations
- Affinity and anti-affinity rules
### Controller Manager (`kube-controller-manager`)
A single binary that runs multiple **control loops** (controllers). Each controller watches the cluster state and works to reconcile the *actual* state with the *desired* state. Examples include:
- **Node Controller** — notices and responds when nodes go down
- **ReplicaSet Controller** — ensures the correct number of pod replicas are always running
- **Deployment Controller** — manages rolling updates
---
 
## 3. Worker Node Components
 
These components run on **every worker node** and handle the actual execution of workloads.
 
### kubelet
The **primary node agent**. It runs on every node and is responsible for:
- Registering the node with the API server
- Receiving Pod specifications (PodSpecs) and ensuring the described containers are running and healthy
- Reporting node and pod status back to the control plane
> **Why swap must be disabled:** The kubelet strictly enforces memory limits. If swap is active, the OS can silently offload memory to disk, making the kubelet's resource accounting unreliable and leading to unpredictable pod behavior.
 
### kube-proxy
A network proxy that runs on each node. It maintains **network rules** that allow communication to Pods from inside or outside the cluster. It implements the Kubernetes `Service` concept at the OS level using iptables or IPVS rules.
 
### Container Runtime
The software responsible for **actually running containers** on a node. Kubernetes communicates with the runtime via the Container Runtime Interface (CRI). In this guide, we use **containerd**.
 
---
 
## 4. Networking Concepts
 
### Pod Network CIDR (`192.168.0.0/16`)
A dedicated block of private IP addresses reserved for **Pods** inside the cluster. When `kubeadm init` is called with `--pod-network-cidr=192.168.0.0/16`, it tells the cluster to hand out IPs from this range to all pods. The `/16` gives 65,536 possible addresses.
 
### CNI — Container Network Interface
Kubernetes has no built-in pod networking. Instead, it defines a standard called **CNI** that third-party plugins implement. The CNI plugin is responsible for:
- Assigning IP addresses to Pods
- Routing traffic between Pods on different nodes
- Enforcing Network Policies
### Calico
The CNI plugin used in this guide. Calico is a production-grade networking solution that uses **BGP (Border Gateway Protocol)** to route traffic between nodes. It is widely used in enterprise environments and is the default recommended plugin for kubeadm setups.
 
### CoreDNS
The **in-cluster DNS server** deployed automatically by kubeadm. It allows Pods and Services to discover each other by name (e.g., `nginx-test.default.svc.cluster.local`) instead of by IP address, which can change.
 
### NodePort
A type of Kubernetes `Service` that exposes an application on a **static port on every node's IP address** (in the range 30000–32767). Any traffic arriving at `<NodeIP>:<NodePort>` is forwarded to the correct Pod, regardless of which node the Pod is actually running on.
 
### Bridge Networking (`br_netfilter`)
A Linux kernel module that allows `iptables` to inspect and filter traffic passing through a **network bridge**. This is essential for Kubernetes because container traffic passes through virtual bridges, and without this module, network policies and routing rules would be bypassed.
 
---
 
## 5. Container Runtime
 
### Docker vs containerd
Kubernetes originally used Docker as its container runtime. In **v1.24**, Kubernetes removed the built-in Docker adapter (Dockershim). The ecosystem moved to **containerd**, which is actually the same runtime that Docker itself uses under the hood — just without the extra Docker layers.
 
### containerd
A lightweight, industry-standard container runtime that implements the CRI. It handles:
- Pulling container images from registries
- Creating and managing container lifecycles
- Interfacing with the OS via `runc`
### cgroup (Control Groups)
A Linux kernel feature that **limits and isolates resource usage** (CPU, memory, disk I/O) for processes. Kubernetes uses cgroups to enforce the resource `requests` and `limits` you set on containers.
 
### cgroup Driver: `systemd`
Both `systemd` (the Linux init system) and container runtimes use cgroups. If they use **different cgroup managers**, resource accounting conflicts and instability can occur. This guide configures containerd to use `systemd` as its cgroup driver — the same as the OS — for a single, unified hierarchy.
 
### Overlay Filesystem (`overlay`)
A Linux kernel module used by container runtimes to implement the **container filesystem**. It stacks multiple filesystem layers (the base image layers + a writable container layer) into a single unified view, making containers efficient in both storage and startup time.
 
---
 
## 6. Key Tools
 
### `kubeadm`
The **official bootstrapping tool** for Kubernetes clusters. It handles the complex task of generating TLS certificates, configuring the API server, and joining nodes to the cluster. It is designed to get a cluster running quickly without requiring deep knowledge of each component's configuration flags.
 
| Command | Purpose |
|---|---|
| `kubeadm init` | Bootstrap the control plane on the master node |
| `kubeadm join` | Attach a worker node to an existing cluster |
| `kubeadm token create` | Generate a new join token (valid 24 hrs) |
 
### `kubectl`
The **command-line interface (CLI)** for interacting with any Kubernetes cluster. It communicates with the API server using a configuration file called `kubeconfig` (located at `~/.kube/config`).
 
| Command | Purpose |
|---|---|
| `kubectl get nodes` | List all nodes and their status |
| `kubectl get pods -n kube-system` | List system pods in the `kube-system` namespace |
| `kubectl create deployment` | Create a new deployment |
| `kubectl expose deployment` | Create a Service to expose a deployment |
| `kubectl get svc` | List all services |
 
### `kubelet`
Not typically invoked directly by users. It runs as a **systemd service** on every node and is the low-level agent that executes the instructions from the control plane.
 
---
 
## 7. Operating System Prerequisites
 
### SELinux
**Security-Enhanced Linux** is a mandatory access control system built into the Linux kernel. In strict (`enforcing`) mode, it can block Kubernetes components from accessing host resources they need. Setting it to `permissive` mode allows operations while still logging violations — a safe middle ground for learning environments.
 
### `firewalld`
The default firewall management daemon on RHEL-based systems. Kubernetes requires many ports to be open between nodes (e.g., `6443` for the API server, `10250` for kubelet). In a lab environment, firewalld is disabled entirely. In production, specific port rules would be configured instead.
 
### Swap
Virtual memory that uses disk space as an overflow for RAM. Kubernetes **requires swap to be disabled** because its memory management and scheduling decisions assume all memory is physical RAM. Unexpected swap usage makes container resource limits unreliable.
 
### `sysctl` Parameters
 
| Parameter | Purpose |
|---|---|
| `net.ipv4.ip_forward = 1` | Allows the Linux kernel to forward packets between network interfaces — essential for routing traffic between pods and nodes |
| `net.bridge.bridge-nf-call-iptables = 1` | Makes iptables rules apply to traffic crossing a network bridge, enabling Kubernetes network policies to work on bridged pod traffic |
| `net.bridge.bridge-nf-call-ip6tables = 1` | Same as above, for IPv6 traffic |
 
---
 
## 8. Kubernetes Objects
 
### Pod
The **smallest deployable unit** in Kubernetes. A Pod wraps one or more containers that share the same network namespace (IP address) and storage. Pods are ephemeral — they can be created, killed, and replaced at any time.
 
### Deployment
A higher-level object that **manages a set of identical Pods**. You tell the Deployment how many replicas you want (e.g., 3 Nginx pods), and the Deployment Controller continuously ensures that number is maintained — restarting pods that crash and replacing pods on failed nodes.
 
```yaml
# Conceptual example of what a Deployment represents
Deployment: nginx-test
  └── ReplicaSet
        ├── Pod (nginx container)
        ├── Pod (nginx container)
        └── Pod (nginx container)
```
 
### Service
An abstraction that provides a **stable network endpoint** for a set of Pods. Because Pod IPs change constantly, a Service gives you a fixed IP or DNS name that automatically routes to healthy pods. Types include:
- **ClusterIP** — only reachable inside the cluster (default)
- **NodePort** — exposes on a port on every node's IP (used in this guide)
- **LoadBalancer** — provisions a cloud load balancer (cloud environments only)
### Namespace
A way to **logically divide** cluster resources into isolated groups. The `kube-system` namespace, for example, contains all the core Kubernetes components (CoreDNS, etcd, API server). User workloads default to the `default` namespace.
 
---
 
## Quick Reference: Port Cheat Sheet
 
| Port | Component | Direction |
|------|-----------|-----------|
| `6443` | API Server | Worker → Master |
| `2379-2380` | etcd | Internal (Master only) |
| `10250` | kubelet API | Master → Worker |
| `10251` | Scheduler | Internal (Master only) |
| `10252` | Controller Manager | Internal (Master only) |
| `30000-32767` | NodePort Services | External → Any Node |
 
---
 
*This reference guide corresponds to the Kubernetes from-scratch installation using `kubeadm` v1.30 and Calico v3.27.3 on CentOS/RHEL 9.*