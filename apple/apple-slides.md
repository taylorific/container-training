---
layout: section
---

# Docker Desktop vs Apple Container

<br>
<br>
<Link to="toc" title="Table of Contents"/>

---
hideInToc: true
layout: two-cols
---

# Docker Desktop

- Commercial product from Docker, Inc.
- Runs **one shared Linux VM** hosting all your containers
- Uses Apple's Virtualization framework (or QEMU) under the hood
- Mature: Compose, Swarm, extensions, CI ecosystem, GUI dashboard
- Works on Intel **and** Apple Silicon Macs
- Free for individuals/small teams; **paid** for orgs over 250 employees or $10M+ revenue

::right::

# Apple Container

- Open-source CLI from Apple (Apache-2.0), written in Swift
- Announced WWDC 2025, reached **v1.0.0** in June 2026
- Runs each container in its **own lightweight VM**
- No daemon, no menu-bar app, no subscription
- **Apple Silicon only** — no Intel support planned
- CLI-only — no bundled GUI

---
hideInToc: true
---

# Architecture: the core difference

<div class="grid grid-cols-2 gap-8 pt-4">

<div>

### Docker Desktop
One big shared Linux VM

```
┌─────────── macOS ───────────┐
│  ┌──────── Linux VM ──────┐ │
│  │ container A            │ │
│  │ container B            │ │
│  │ container C            │ │
│  │ (shared kernel)        │ │
│  └────────────────────────┘ │
└─────────────────────────────┘
```

Containers share one kernel and one VM's resource pool.

</div>

<div>

### Apple Container
One micro-VM per container

```
┌──────────── macOS ────────────┐
│ ┌───VM A───┐ ┌───VM B───┐     │
│ │container │ │container │ ... │
│ │   A      │ │   B      │     │
│ └──────────┘ └──────────┘     │
└───────────────────────────────┘
```

Each container gets hypervisor-level isolation via `Virtualization.framework`.

</div>

</div>

---
hideInToc: true
---

# Why the architecture matters

<v-clicks>

- **Isolation**: Apple's per-container VMs give hardware-level separation between workloads — a compromised container can't as easily see its neighbors
- **Resource overhead**: Docker's shared VM amortizes memory/CPU better across many small containers; Apple spins up a VM per container, which adds baseline overhead per container
- **Blast radius**: a kernel-level bug affects one container under Apple's model, vs. potentially all containers sharing Docker's VM
- **Isolation vs. density** is the real trade-off — pick based on whether you're running 3 containers or 30

</v-clicks>

---
hideInToc: true
---

# Biggest difference is in networking

<div class="grid grid-cols-2 gap-8 pt-4">

<div>

### Docker Desktop
- Containers sit behind the shared VM
- Requires **port mapping** (`-p 8080:80`) to reach a container from the Mac
- Supports `--network host`, `--link`, custom bridge networks

</div>

<div>

### Apple Container
- Each container gets its **own IP address**
- Reachable directly by name: `myapp.dev.internal` — no port mapping needed
- **No** `--network host` or `--link` yet
- Cross-container networking still has rough edges; isolated per-container networks need macOS 26 (Tahoe) — macOS 15 forces a shared network

</div>

</div>

---
layout: section
---

# Apple Containerization on macOS

<br>
<br>
<Link to="toc" title="Table of Contents"/>

---
hideInToc: true
---

# container

`container` is Apple's command-line tool for running Linux containers on macOS.

You need a mac with Apple silicon and macOS 26 (Tahoe) or later to run `container`.

`container` does not come with macOS, you must download the latest signed
installer from https://github.com/apple/container/releases

---
hideInToc: true
---

# Check requirements

Check to make sure that your machine is an Apple Silicon mac.

```bash
# Architecture should be arm64
% uname -m
arm64
```

And make sure you are running at least macOS 26 (Tahoe).

```bash
# Should be at least ProductVersion 26
% sw_vers
ProductName:		macOS
ProductVersion:		26.6.2
BuildVersion:		25G83
```

---

# Installing container with gh

If you have the GitHUB cli installed:

```bash
mkdir -p /tmp/apple-container
cd /tmp/apple-container

gh release download \
  --repo apple/container \
  --pattern '*installer-signed.pkg'

sudo installer \
  -pkg container-*-installer-signed.pkg \
  -target /
```

---

# Installing container with curl

If you do not have the GitHUB cli installed, you can also download the
installer entirely with `curl`, using the GitHub API.

```bash
mkdir -p /tmp/apple-container
cd /tmp/apple-container

url=$(
  curl -s https://api.github.com/repos/apple/container/releases/latest |
  jq -r '.assets[]
         | select(.name | endswith("installer-signed.pkg"))
         | .browser_download_url'
)

curl -L -O "$url"

sudo installer \
  -pkg container-*-installer-signed.pkg \
  -target /
```

---

# Uninstalling container

To uninstall the Apple Container CLI tool, use the uninstall script
located in `/usr/local/bin`.

```bash
# Keep User Data
/usr/local/bin/uninstall-container.sh -k

# Delete all user data
/usr/local/bin/uninstall-container.sh -d
```

---
hideInToc: true
---

# Updating container

If you're updating container, first stop your existing container daemon:

```bash
container system stop
```

To upgrade to the latest release, use the `update-container.sh` script (in `/usr/local/bin`):

```
/usr/local/bin/update-container.sh
```

To downgrade, uninstall your existing container and use the `-v` flag to install a specific version:

```
# -k flag keeps user data, -d deletes all user data
/usr/local/bin/uninstall-container.sh -k
/usr/local/bin/update-container.sh -v 1.4.0
```

After updating, start the system service again with:

```bash
container system start
```

---
hideInToc: true
---

# Starting the container service

```bash
 % container system start
Launching container-apiserver...
Testing access to container-apiserver...
Verifying machine API server is running...
No default kernel configured.
Install the recommended default kernel from [https://github.com/kata-containers/kata-containers/releases/download/3.32.0/kata-static-3.32.0-arm64.tar.zst]? [Y/n]:
Installing kernel...
```

---
hideInToc: true
---

# Running your first Apple container

```bash
container run --rm alpine echo hello
```

---
hideInToc: true
---

# container tool pulls images from Docker Hub by default

By default, Apple's `container` tool uses **Docker Hub (docker.io)** for
unqualified image names.

To view the current setting:

```bash
container system property get registry.domain
```

To change it:

```bash
container property set registry.domain registry.example.com
```

To revert to the built-in default:

```bash
container system property clear registry.domain
```

---
hideInToc: true
---

# container settings

To see all current properties:

```bash
contaienr system property list
```

The configuration is effectively immutable while the service is running.
The TOML files are read once at service startup.

The service reads configuration with first-match-wins precedence:

```bash
1. ~/.config/container/config.toml
           ↓
2. /usr/local/etc/container/config.toml
           ↓
3. hard-coded defaults
```

---
hideInToc: true
---

# Container Networking

Containerization provides networking services for containers.

Starting the system creates a default network.

Inspect it:

```bash
container network list
```

You can also create isolated networks:

```bash
container network create lab
```

Then attach a workload:

```bash
container run \
  --network lab \
  --rm \
  -it alpine sh
```

---
hideInToc: true
---

# Resource Isolation

Because each container is backed by a lightweight VM,
resources can be assigned directly to that environment.

For example:

```bash
container run \
  --rm \
  --cpus 2 \
  --memory 2g \
  ubuntu:latest
```

Think about what this means.

Instead of:

```text
cgroup limit
     │
     ▼
shared kernel
```

we now also have:

```text
VM resource allocation
     │
     ▼
Linux kernel
     │
     ▼
container workload
```

---
hideInToc: true
---

# Images Are Still OCI Images

Apple did not invent a new container image format.

The system works with standard:

# OCI images

For example:

```bash
container image pull ubuntu:latest
```

List images:

```bash
container image list
```

Run one:

```bash
container run --rm -it ubuntu:latest bash
```

The image ecosystem remains familiar.

---
hideInToc: true
---

# Build Images

Apple's CLI can also build container images.

For example:

```dockerfile
FROM alpine

RUN apk add --no-cache curl

CMD ["sh"]
```

Build:

```bash
container build -t demo .
```

Run:

```bash
container run --rm -it demo
```

So the developer workflow remains recognizable.

---
hideInToc: true
---

# What About Persistent Linux Environments?

A container is usually modeled around:

```text
an application
```

But developers sometimes want:

```text
a Linux development machine
```

Apple now provides another abstraction:

# Container Machine

---
hideInToc: true
---

# Container Machine

A Container Machine is:

```text
fast
lightweight
persistent
Linux
OCI-image based
integrated with macOS
```

Think:

```text
container
    =
ephemeral application environment
```

versus:

```text
container machine
    =
persistent Linux environment
```

---
hideInToc: true
---

# Create a Container Machine

For example:

```bash
container machine create \
  --name demo \
  --set-default \
  alpine
```

Then:

```bash
container machine run
```

You now have an interactive Linux environment.


---
hideInToc: true
---

# Sources

Apple Container: https://github.com/apple/container

Apple Containerization: https://github.com/apple/containerization

WWDC25: Meet Containerization

WWDC26: Discover container machines

Apple Container Technical Overview: https://github.com/apple/container/blob/main/docs/technical-overview.md

Apple Container: Best Guide for 2026: https://kalinga.ai/apple-container-guide-2026/

Apple Containers on macOS: A Technical Comparison With Docker: https://thenewstack.io/apple-containers-on-macos-a-technical-comparison-with-docker/

Tired of updating Docker for Mac apple/container is enough: https://www.outcoldman.com/blog/2026/05/02/apple-container-tired-of-docker/
