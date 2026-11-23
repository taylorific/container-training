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
layout: section
---

# Talos

An operating system that only runs Kubernetes

<br>
<br>
<Link to="toc" title="Table of Contents"/>

---
hideInToc: true
layout: two-cols
---

# Talos: what it is

- A minimal, **immutable Linux OS** built only to run Kubernetes
- **No SSH, no shell, no package manager**
- Managed entirely through an API with `talosctl` and a declarative machine config
- Upgrades replace the whole OS image instead of patching it
- From Sidero Labs

::right::

# When to use it

- You want nodes defined by config files you can check in, not by hand-run commands
- Bare metal, VMs and homelabs
- A small attack surface: nothing on the node to log in to

Explore it once you're comfortable with kubeadm, so you know what Talos is doing for you.

---
hideInToc: true
---

# Talos: build an install image

The **Image Factory** at https://factory.talos.dev/ builds an image with the extensions you need:

- Hardware type: **Bare-metal Machine**; latest stable Talos version; architecture **amd64**
- System extensions: `qemu-guest-agent`, `util-linux-tools`
- Extra kernel arguments: `console=ttyS0,115200`; **UEFI only**

At "schematic ready", download the ISO:

```bash
mkdir talos-cluster && cd talos-cluster
SCHEMATIC=9385ba106a3a7d3eff277f44bf0468b6de749769beee4ff47951f31b4955d334
curl -LO https://factory.talos.dev/image/$SCHEMATIC/v1.12.3/metal-amd64.iso
shasum -a 256 metal-amd64.iso
sudo cp metal-amd64.iso /var/lib/libvirt/iso/metal-amd64.iso
```

The ISO boots into a **RAM-only live mode**; Talos installs itself to disk once you apply a config.

<!--
The long path in the URL is the schematic ID: it identifies the exact set of extensions and kernel arguments chosen in the Image Factory. Check the Talos version before each delivery.
-->

---
hideInToc: true
---

# Talos: boot a node (KVM)

```bash
# UEFI firmware: the image was built UEFI only
LOADER=loader=/usr/share/OVMF/OVMF_CODE_4M.fd,loader.readonly=yes,loader.type=pflash
NVRAM=nvram.template=/usr/share/OVMF/OVMF_VARS_4M.fd

virt-install \
  --connect qemu:///system \
  --name talos-cluster \
  --boot $LOADER,$NVRAM \
  --cdrom /var/lib/libvirt/iso/metal-amd64.iso \
  --disk pool=default,format=qcow2,bus=virtio,size=64 \
  --memory 4096 \
  --vcpus 4 \
  --os-variant linux2022 \
  --network network=host-network,model=virtio \
  --graphics spice \
  --noautoconsole

# Talos boots into RAM-only live mode
virt-viewer talos-cluster &
```

Any VM or bare-metal machine works the same way: boot the ISO and note the IP address on its console.

---
hideInToc: true
---

# Talos: install talosctl

Use the **same version** as the image:

```bash
curl -LO https://github.com/siderolabs/talos/releases/download/v1.12.3/talosctl-linux-amd64
shasum -a 256 talosctl-linux-amd64
sudo install -m 0755 talosctl-linux-amd64 /usr/local/bin/talosctl

talosctl version
```

```text
Client:
	Tag:         v1.12.3
Server:
error constructing client: failed to determine endpoints
```

The server error is expected: there's no cluster to talk to yet.

---
hideInToc: true
---

# Talos: find the node and its disk

```bash
# IP address from qemu-guest-agent, or read it from the console
virsh domifaddr talos-cluster --source agent
export CONTROL_PLANE_IP=10.63.34.66

talosctl get disks --insecure --nodes "$CONTROL_PLANE_IP"
```

```text
NODE   NAMESPACE   TYPE   ID      VERSION   SIZE     READ ONLY   TRANSPORT   ROTATIONAL   MODEL
       runtime     Disk   sr0     2         215 MB   false       sata        true         QEMU DVD-ROM
       runtime     Disk   vda     2         69 GB    false       virtio      true
```

```bash
export DISK_ID=vda
```

`--insecure` is only for this first contact, before the node has a config and certificates.

---
hideInToc: true
---

# Talos: generate and apply the config

```bash
# Generate the cluster's machine configs and client config
talosctl gen config talos-cluster https://$CONTROL_PLANE_IP:6443 \
  --install-disk /dev/$DISK_ID -o configs/

# A single-node cluster must also run workloads on the control plane
cat > single-node-cluster-patch.yaml <<EOF
cluster:
  allowSchedulingOnControlPlanes: true
EOF

# Apply it: the node installs to disk and reboots
talosctl apply-config --insecure \
  --nodes $CONTROL_PLANE_IP \
  --file configs/controlplane.yaml \
  --config-patch @single-node-cluster-patch.yaml
```

`configs/` now holds `controlplane.yaml`, `worker.yaml` and `talosconfig`: the whole cluster as files.

---
hideInToc: true
---

# Talos: bootstrap the cluster

```bash
# Bootstrap etcd, once, on one control plane node
talosctl bootstrap --nodes $CONTROL_PLANE_IP -e $CONTROL_PLANE_IP \
  --talosconfig configs/talosconfig

# Watch it come up
talosctl dashboard --nodes $CONTROL_PLANE_IP -e $CONTROL_PLANE_IP \
  --talosconfig configs/talosconfig

talosctl logs -f -n $CONTROL_PLANE_IP -e $CONTROL_PLANE_IP \
  --talosconfig configs/talosconfig etcd
talosctl service -n $CONTROL_PLANE_IP -e $CONTROL_PLANE_IP \
  --talosconfig configs/talosconfig
```

There's no shell on the node: logs, services and the dashboard all come through the API.

---
hideInToc: true
---

# Talos: use the cluster

```bash
# kubectl within one minor version of the cluster (Kubernetes v1.35.0 here)
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

talosctl kubeconfig ./kubeconfig -n $CONTROL_PLANE_IP -e $CONTROL_PLANE_IP \
  --talosconfig configs/talosconfig
export KUBECONFIG=./kubeconfig
kubectl get nodes
```

```text
NAME            STATUS   ROLES           AGE     VERSION
talos-j06-rk3   Ready    control-plane   8m54s   v1.35.0
```

`kubectl get pods -A` shows CoreDNS, Flannel, kube-proxy and the control plane: Talos chose the CNI for you.

---
hideInToc: true
---

# Talos: clean up and read more

```bash
virsh destroy talos-cluster
virsh undefine talos-cluster --nvram --remove-all-storage
```

References:

- Talos on KVM: https://docs.siderolabs.com/talos/v1.11/platform-specific-installations/virtualized-platforms/kvm
- Talos with Cilium on Proxmox: https://unixorn.github.io/post/homelab/k8s/01-talos-with-cilium-cni-on-proxmox/

---
hideInToc: true
---

# Or skip it: managed Kubernetes

- **EKS**, **GKE**, **AKS**
- The provider runs the control plane: upgrades, etcd, certificates
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
