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

# Starting the container daemon

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

# Start an interative environment

```bash
container run --rm -it alpine sh
```

Investigate the environment:

```bash
uname -a
cat /etc/os-release
ps aux
ip addr
mount
```

---
hideInToc: true
layout: two-cols
---

# Apple Container

```bash
% container run --rm -it alpine sh
/ # uname -a
Linux 76d483c5-9e07-4ba5-baab-72bf6cce0a8c 6.18.35 #1 SMP Mon Jun 15 12:55:27 UTC 2026 aarch64 Linux
/ # cat /etc/os-release
NAME="Alpine Linux"
ID=alpine
VERSION_ID=3.24.1
PRETTY_NAME="Alpine Linux v3.24"
HOME_URL="https://alpinelinux.org/"
BUG_REPORT_URL="https://gitlab.alpinelinux.org/alpine/aports/-/issues"
/ # ps aux
PID   USER     TIME  COMMAND
    1 root      0:00 sh
    4 root      0:00 ps aux
```

::right::

# Docker

```bash
% docker container run -it --rm alpine sh
/ # uname -a
Linux ced167defe6f 6.10.14-linuxkit #1 SMP Wed Sep 10 06:47:45 UTC 2025 aarch64 Linux
/ # cat /etc/os-release
NAME="Alpine Linux"
ID=alpine
VERSION_ID=3.24.1
PRETTY_NAME="Alpine Linux v3.24"
HOME_URL="https://alpinelinux.org/"
BUG_REPORT_URL="https://gitlab.alpinelinux.org/alpine/aports/-/issues"
/ # ps aux
PID   USER     TIME  COMMAND
    1 root      0:00 sh
    9 root      0:00 ps aux
```

---
hideInToc: true
layout: two-cols
---

# Apple Container

```bash
% container run --rm -it alpine sh
/ # ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,UP,LOWER_UP> mtu 1280 qdisc pfifo_fast state UP qlen 1000
    link/ether fa:7e:6c:56:d3:e6 brd ff:ff:ff:ff:ff:ff
    inet 192.168.64.4/24 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fd44:bd2:73c8:6de3:f87e:6cff:fe56:d3e6/64 scope global dynamic flags 100
       valid_lft 2591980sec preferred_lft 604780sec
    inet6 fe80::f87e:6cff:fe56:d3e6/64 scope link
       valid_lft forever preferred_lft forever
```

::right::

# Docker

```bash
% docker container run -it --rm alpine sh
/ # ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: tunl0@NONE: <NOARP> mtu 1480 qdisc noop state DOWN qlen 1000
    link/ipip 0.0.0.0 brd 0.0.0.0
3: gre0@NONE: <NOARP> mtu 1476 qdisc noop state DOWN qlen 1000
    link/gre 0.0.0.0 brd 0.0.0.0
4: gretap0@NONE: <BROADCAST,MULTICAST> mtu 1462 qdisc noop state DOWN qlen 1000
    link/ether 00:00:00:00:00:00 brd ff:ff:ff:ff:ff:ff
5: erspan0@NONE: <BROADCAST,MULTICAST> mtu 1450 qdisc noop state DOWN qlen 1000
    link/ether 00:00:00:00:00:00 brd ff:ff:ff:ff:ff:ff
6: ip_vti0@NONE: <NOARP> mtu 1480 qdisc noop state DOWN qlen 1000
    link/ipip 0.0.0.0 brd 0.0.0.0
7: ip6_vti0@NONE: <NOARP> mtu 1428 qdisc noop state DOWN qlen 1000
    link/tunnel6 00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00 brd 00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00
8: sit0@NONE: <NOARP> mtu 1480 qdisc noop state DOWN qlen 1000
    link/sit 0.0.0.0 brd 0.0.0.0
9: ip6tnl0@NONE: <NOARP> mtu 1452 qdisc noop state DOWN qlen 1000
    link/tunnel6 00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00 brd 00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00
10: ip6gre0@NONE: <NOARP> mtu 1448 qdisc noop state DOWN qlen 1000
    link/[823] 00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00 brd 00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00
11: eth0@if22: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 65535 qdisc noqueue state UP
    link/ether b6:9b:e6:32:84:c3 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.3/16 brd 172.17.255.255 scope global eth0
       valid_lft forever preferred_lft forever
```

---
hideInToc: true
layout: two-cols
---

# Apple Container

```bash
% container run --rm -it alpine sh
/ # mount
/dev/vdb on / type ext4 (rw,relatime)
proc on /proc type proc (rw,relatime)
sysfs on /sys type sysfs (rw,nosuid,nodev,noexec,relatime)
none on /dev type devtmpfs (rw,nosuid,relatime,size=561360k,nr_inodes=140340,mode=755)
mqueue on /dev/mqueue type mqueue (rw,nosuid,nodev,noexec,relatime)
tmpfs on /dev/shm type tmpfs (rw,nosuid,nodev,noexec,relatime,size=65536k)
devpts on /dev/pts type devpts (rw,nosuid,noexec,relatime,gid=5,mode=620,ptmxmode=666)
none on /sys/fs/cgroup type cgroup2 (rw,nosuid,nodev,noexec,relatime)
devpts on /dev/console type devpts (rw,nosuid,noexec,relatime,gid=5,mode=620,ptmxmode=666)
none on /proc/keys type devtmpfs (rw,nosuid,relatime,size=561360k,nr_inodes=140340,mode=755)
none on /proc/timer_list type devtmpfs (rw,nosuid,relatime,size=561360k,nr_inodes=140340,mode=755)
tmpfs on /sys/firmware type tmpfs (ro,nosuid,nodev,noexec,relatime)
proc on /proc/bus type proc (ro,relatime)
proc on /proc/fs type proc (ro,relatime)
proc on /proc/irq type proc (ro,relatime)
proc on /proc/sys type proc (ro,relatime)
```

::right::

# Docker

```bash
% docker container run -it --rm alpine sh
/ # mount
overlay on / type overlay (rw,relatime,lowerdir=/var/lib/desktop-containerd/daemon/io.containerd.snapshotter.v1.overlayfs/snapshots/3427/fs:/var/lib/desktop-containerd/daemon/io.containerd.snapshotter.v1.overlayfs/snapshots/3423/fs,upperdir=/var/lib/desktop-containerd/daemon/io.containerd.snapshotter.v1.overlayfs/snapshots/3428/fs,workdir=/var/lib/desktop-containerd/daemon/io.containerd.snapshotter.v1.overlayfs/snapshots/3428/work)
proc on /proc type proc (rw,nosuid,nodev,noexec,relatime)
tmpfs on /dev type tmpfs (rw,nosuid,size=65536k,mode=755)
devpts on /dev/pts type devpts (rw,nosuid,noexec,relatime,gid=5,mode=620,ptmxmode=666)
sysfs on /sys type sysfs (ro,nosuid,nodev,noexec,relatime)
cgroup on /sys/fs/cgroup type cgroup2 (ro,nosuid,nodev,noexec,relatime)
mqueue on /dev/mqueue type mqueue (rw,nosuid,nodev,noexec,relatime)
shm on /dev/shm type tmpfs (rw,nosuid,nodev,noexec,relatime,size=65536k)
/dev/vda1 on /etc/resolv.conf type ext4 (rw,relatime,discard)
/dev/vda1 on /etc/hostname type ext4 (rw,relatime,discard)
/dev/vda1 on /etc/hosts type ext4 (rw,relatime,discard)
devpts on /dev/console type devpts (rw,nosuid,noexec,relatime,gid=5,mode=620,ptmxmode=666)
proc on /proc/bus type proc (ro,nosuid,nodev,noexec,relatime)
proc on /proc/fs type proc (ro,nosuid,nodev,noexec,relatime)
proc on /proc/irq type proc (ro,nosuid,nodev,noexec,relatime)
proc on /proc/sys type proc (ro,nosuid,nodev,noexec,relatime)
proc on /proc/sysrq-trigger type proc (ro,nosuid,nodev,noexec,relatime)
tmpfs on /proc/interrupts type tmpfs (rw,nosuid,size=65536k,mode=755)
tmpfs on /proc/kcore type tmpfs (rw,nosuid,size=65536k,mode=755)
tmpfs on /proc/keys type tmpfs (rw,nosuid,size=65536k,mode=755)
tmpfs on /proc/timer_list type tmpfs (rw,nosuid,size=65536k,mode=755)
tmpfs on /proc/scsi type tmpfs (ro,relatime)
tmpfs on /sys/firmware type tmpfs (ro,relatime)
```
---
hideInToc: true
---

# Look at the Isolation

Start two containers.

Terminal 1:

```bash
container run --name one -it alpine sh
```

Terminal 2:

```bash
container run --name two -it alpine sh
```

Inside each:

```bash
ps aux
ip addr
mount
```

Think about what you are seeing.

These are not two namespaces inside one conventional
Docker-style Linux VM.

Each workload has VM-based isolation.