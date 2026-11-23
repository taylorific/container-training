---
layout: section
---

# Installing Kubernetes

<br>
<br>
<Link to="toc" title="Table of Contents"/>

---
hideInToc: true
---

# Why are there so many options?

Kubernetes standardized how applications **run** on a cluster — but not how you
**build, install, or operate** that cluster.

<v-clicks>

1. **Designed for large-scale infrastructure** — born from Google's Borg ideas to manage fleets of machines, not one laptop
2. **Upstream isn't a turnkey product** — you still choose a container runtime, networking (CNI), storage (CSI), ingress, load balancing, certificates, upgrades, and OS
3. **Different audiences, different needs** — a developer wants a cluster in 20 seconds; edge/robotics wants low resource use; infra engineers want predictable upgrades and security; cloud providers want integration
4. **Vendor differentiation** — Canonical (MicroK8s), SUSE/Rancher (k3s), Docker, Sidero Labs (Talos)
5. **Open source makes alternatives easy, consolidation hard** — once a tool has users, docs, and maintainers, it persists

</v-clicks>

---
hideInToc: true
---

# There are really only three approaches

<div class="grid grid-cols-3 gap-6 pt-4">

<div>

### 1. Fake the machines with containers

`kind`, `k3d`

Fast, disposable clusters for development and testing.

</div>

<div>

### 2. Package Kubernetes for easy install

`k3s`, `MicroK8s`, `minikube`

Simpler to install and use by supplying sensible defaults.

</div>

<div>

### 3. Build and manage real infrastructure

`kubeadm`, `Talos`

You take responsibility for nodes, OS, and cluster lifecycle.

</div>

</div>

<br>

The categories overlap: k3s runs on real machines, minikube can use containers
**or** VMs, and Talos is an operating system rather than a Kubernetes distribution.

---
hideInToc: true
---

# Recommendation: ignore 80% of the options

<div class="grid grid-cols-3 gap-6 pt-4">

<div>

### kind
**Disposable dev clusters**

Kubernetes nodes are containers. Create, test, and throw away in seconds.

</div>

<div>

### k3s
**Lightweight real deployments**

A single binary with batteries included. Runs on VMs, servers, edge devices, and Raspberry Pis.

</div>

<div>

### kubeadm
**Understand the machinery**

The upstream bootstrapping tool. You assemble the runtime, networking, and nodes yourself.

</div>

</div>

<br>

Add **Talos** later when you want to explore declarative, immutable node management.

---
layout: section
---

# kind

Kubernetes IN Docker

<br>
<br>
<Link to="toc" title="Table of Contents"/>

---
hideInToc: true
layout: two-cols
---

# kind: what it is

- Each Kubernetes **node is a container** running on your container engine
- Created to **test Kubernetes itself**
- Runs a real upstream control plane (built with `kubeadm` inside the node container)
- Multi-node clusters on a single laptop
- Clusters start in under a minute and are **disposable**

::right::

# When to use it

- Local development and testing
- CI pipelines that need a real cluster
- Trying out manifests, Helm charts, operators
- Learning `kubectl` without risk

**Not** for production — no persistence guarantees, and all "nodes" share one host

---
hideInToc: true
---

# Installing kind

Requires a container engine: Docker, Podman, or nerdctl.

```bash
# macOS / Linux with Homebrew
brew install kind

# Or with Go
go install sigs.k8s.io/kind@latest

# You'll also need kubectl
brew install kubectl
```

Verify:

```bash
kind version
```

---
hideInToc: true
---

# Creating a kind cluster

```bash
# Single-node cluster
kind create cluster --name dev

# kind sets your kubectl context automatically
kubectl cluster-info --context kind-dev
kubectl get nodes
```

Multi-node cluster with a config file:

```yaml
# kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
```

```bash
kind create cluster --name multi --config kind-config.yaml
kind get clusters
kind delete cluster --name multi
```

---
layout: section
---

# k3s

Lightweight Kubernetes

<br>
<br>
<Link to="toc" title="Table of Contents"/>

---
hideInToc: true
layout: two-cols
---

# k3s: what it is

- CNCF project originally created by Rancher (now SUSE)
- Fully conformant Kubernetes in a **single binary**
- Low memory footprint — fits on edge devices
- Runs on **real machines**: VMs, bare metal, ARM boards
- Linux only (use a VM on macOS/Windows, or `k3d` to run k3s in containers)

::right::

# Batteries included

Sensible defaults chosen for you:

- **Runtime**: containerd
- **CNI**: Flannel
- **Ingress**: Traefik
- **Load balancer**: ServiceLB
- **Storage**: local-path-provisioner
- **DNS**: CoreDNS
- **Datastore**: SQLite (single node) or embedded etcd (HA)

Any of these can be disabled and swapped out.

---
hideInToc: true
---

# Installing k3s: server

On a Linux machine:

```bash
curl -sfL https://get.k3s.io | sh -
```

This installs k3s as a systemd service, plus `kubectl`, `crictl`, and `ctr`.

```bash
# Check the cluster
sudo k3s kubectl get nodes

# kubeconfig lives here
sudo cat /etc/rancher/k3s/k3s.yaml
```

To use a regular `kubectl`, copy `/etc/rancher/k3s/k3s.yaml` to `~/.kube/config`
and replace `127.0.0.1` with the server's IP if connecting remotely.

---
hideInToc: true
---

# Installing k3s: adding agent nodes

Get the join token from the server:

```bash
sudo cat /var/lib/rancher/k3s/server/node-token
```

On each worker (agent) node:

```bash
curl -sfL https://get.k3s.io | \
  K3S_URL=https://<server-ip>:6443 \
  K3S_TOKEN=<node-token> \
  sh -
```

Back on the server:

```bash
sudo k3s kubectl get nodes
```

Uninstall with `/usr/local/bin/k3s-uninstall.sh` (server) or
`/usr/local/bin/k3s-agent-uninstall.sh` (agent).

---
layout: section
---

# kubeadm

The upstream bootstrapping tool

<br>
<br>
<Link to="toc" title="Table of Contents"/>

---
hideInToc: true
layout: two-cols
---

# kubeadm: what it is

- Official Kubernetes project tool for bootstrapping clusters
- Sets up the control plane: API server, etcd, scheduler, controller manager
- Handles certificates, join tokens, and upgrades (`kubeadm upgrade`)
- The foundation many other tools build on (kind uses it internally)

::right::

# Your responsibility

kubeadm deliberately **does not** choose for you:

- Operating system and kernel settings
- Container runtime (e.g. containerd)
- Pod network / CNI (Calico, Cilium, Flannel…)
- Storage (CSI), ingress, load balancing
- Provisioning the machines themselves

This is why it's the best way to **understand the machinery**.

---
hideInToc: true
---

# kubeadm: prepare every node

```bash
# Kernel modules and networking sysctls
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sudo sysctl --system

# Disable swap (kubelet expects it off by default)
sudo swapoff -a

# Install a container runtime (containerd) and configure the systemd cgroup driver
```

Then install `kubelet`, `kubeadm`, and `kubectl` from the official
package repository at `pkgs.k8s.io`.

---
hideInToc: true
---

# kubeadm: create the cluster

On the control plane node:

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16

# Configure kubectl for your user
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Install a pod network (CNI) — nodes stay NotReady until you do
kubectl apply -f <your-cni-manifest.yaml>
```

On each worker node, run the join command printed by `kubeadm init`:

```bash
sudo kubeadm join <control-plane-ip>:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

---
hideInToc: true
layout: two-cols
---

# Next step: Talos

- A minimal, **immutable Linux OS** built only to run Kubernetes
- No SSH, no shell, no package manager
- Managed via API (`talosctl`) and declarative config
- From Sidero Labs

Explore it once you're comfortable with kubeadm and want declarative node management.

::right::

# Or: managed Kubernetes

- **EKS**, **GKE**, **AKS**
- The provider runs the control plane
- Removes much of the cluster-management burden
- Does **not** remove the complexity of running workloads

---
hideInToc: true
---

# The architectural irony

Kubernetes was supposed to provide a consistent abstraction over infrastructure...

...yet its installation and operations ecosystem became another substantial
layer of infrastructure complexity.

<br>

<v-click>

### The good news

Once a cluster is running, the **Kubernetes API**, **workload objects**, and
**`kubectl` experience** are largely the same.

You don't need to learn ten different Kubernetes systems. You need to learn
Kubernetes — and understand which installation decisions each tool makes for you.

</v-click>
