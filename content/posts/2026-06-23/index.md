---
title: "2026-06-23"
description: "Installing k3s"
date: 2026-06-23T11:15:00+02:00
math: false
license: "CC BY-NC-SA 4.0"
hidden: false
comments: true
draft: false
tags:
    - Software
    - Kubernetes
categories:
    - Software
---
# Installing k3s

> Warning:
>
> This article describes the preparation, installation, and initial configuration of a Kubernetes cluster.
>
> You are reading at your own risk. There is a non-zero chance that your life will never be quite the same afterward.
>
> All procedures described here have been tested.
>
> Also, no animals were harmed during the writing of this article. Unless we count two flies, six mosquitoes, and one unfortunate snail removed from a home cucumber plantation.

# Hardware

The cluster consists of the following machines:

| Hostname  | IP Address  | SBC              | Architecture | RAM   | Storage | Operating System | Intended Role |
| --------- | ----------- | ---------------- | ------------ | ----- | ------- | ---------------- | ------------- |
| palladium | 10.10.10.20 | Odroid N2        | arm64        | 4 GB  | 128 GB  | Armbian          | tiny-int      |
| vanadium  | 10.10.10.21 | Raspberry Pi 5   | arm64        | 8 GB  | 512 GB  | Ubuntu Server    | ci-cd         |
| iridium   | 10.10.10.24 | Odroid M2        | arm64        | 16 GB | 512 GB  | Armbian          | control-plane |
| ruthenium | 10.10.10.25 | Orange Pi 5 Plus | arm64        | 16 GB | 512 GB  | Armbian          | heavy-worker  |
| tantalum  | 10.10.10.26 | Raspberry Pi 500 | arm64        | 16 GB | 256 GB  | Ubuntu Server    | observability |

All IP addresses are assigned by the home router using static DHCP reservations based on MAC addresses.

# Operating System and Required Software

All nodes run either Ubuntu Server or Armbian based on Ubuntu Noble.

Versions may differ slightly, but package management remains identical across the cluster.

The first step is ensuring that every machine can resolve the names of every other machine.

At the moment there is no local DNS server, so each host requires an `/etc/hosts` entry similar to the following:

```text
10.10.10.20 palladium
10.10.10.21 vanadium
10.10.10.24 iridium
10.10.10.25 ruthenium
10.10.10.26 tantalum
```

It is also worth adding the same entries to the workstation or laptop used to manage the cluster.

Windows users may perform the traditional rite of passage and figure out where Microsoft decided to hide the hosts file this decade.

## Useful Tools

It is highly recommended to install `kubectl` on your workstation.

The official installation instructions can be found here:

https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/

On every cluster node, install a basic set of utilities:

```bash
sudo apt update && sudo apt install -y \
  curl \
  wget \
  vim \
  git \
  htop \
  net-tools \
  iproute2 \
  iptables \
  ca-certificates \
  gnupg \
  lsb-release \
  jq \
  socat \
  conntrack \
  open-iscsi \
  nfs-common \
  tmux \
  dnsutils \
  tcpdump \
  lsof \
  nc
```

If you prefer the Emacs lifestyle but do not enjoy "the Sixth Editor", replace `vim` with `mg`.

You may also want:

```bash
sudo apt install mc
```

if navigating directories through a dual-panel interface reminds you of happier times.

On `iridium`, I additionally install:

```bash
sudo apt install -y sqlite3
```

Swap was already disabled during the preparation of the machines, but it is worth double-checking before proceeding.

## Kernel Modules

Enable the required kernel modules:

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k3s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
```

Then configure the required kernel parameters:

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k3s.conf
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
net.ipv4.ip_forward=1
EOF

sudo sysctl --system
```

## cgroups

It is also worth checking whether cgroups are properly enabled:

```bash
cat /proc/cgroups
```

Some systems may require additional kernel parameters in `/boot/cmdline.txt`:

```text
cgroup_enable=cpuset cgroup_enable=memory cgroup_memory=1
```

# Kubernetes

The chosen distribution is **k3s**.

It is fully compatible with "serious" Kubernetes while requiring significantly fewer resources, making it an excellent choice for home labs, edge computing, and IoT deployments.

Small, lightweight, easy to install, and surprisingly capable. What more could one ask for?

## Control Plane

`iridium` has been selected as the control-plane node because it is the most powerful machine in the cluster.

The remaining nodes will join as workers with specific responsibilities.

Install k3s on `iridium`:

```bash
curl -sfL https://get.k3s.io | sh -s - server \
  --node-name iridium \
  --node-ip 10.10.10.24 \
  --advertise-address 10.10.10.24 \
  --write-kubeconfig-mode 644 \
  --disable traefik
```

Run this command directly on the machine, either through SSH or a local terminal session.

### What Do These Flags Do?

* `--node-ip` — forces a specific IP address and therefore a specific network interface.
* `--advertise-address` — makes the API server reachable from the LAN.
* `--disable traefik` — k3s ships with Traefik as its default ingress controller, but this cluster will use NGINX later.
* `--write-kubeconfig-mode 644` — creates the kubeconfig file with world-readable permissions.

That's it.

Really.

After installation, verify that everything is running correctly:

```bash
sudo systemctl status k3s
```

More importantly:

```bash
kubectl get nodes
```

should produce something similar to:

```text
iridium   Ready   control-plane
```

An even better command is:

```bash
kubectl get nodes -w
```

The `-w` option continuously watches for changes.

This is useful because a node that repeatedly transitions between `Ready` and some other state usually indicates an incomplete kernel configuration.

The most common culprits are missing `overlay` or `br_netfilter` modules.

If everything looks healthy, retrieve the cluster configuration:

```bash
cat /etc/rancher/k3s/k3s.yaml
```

Copy it to your workstation.

You may store it as:

```text
~/.kube/config
```

or keep multiple cluster configurations separately.

For Bash:

```bash
export KUBECONFIG=~/.kube/my-cluster-config
```

For Fish:

```fish
set -x KUBECONFIG ~/.kube/my-cluster-config
```

Before using the file, edit the server address:

```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: ...
    server: https://127.0.0.1:6443
```

Replace:

```text
127.0.0.1
```

with:

```text
10.10.10.24
```

You should now be able to run:

```bash
kubectl get nodes
```

directly from your workstation and obtain the same results.

## Joining Worker Nodes

The node registration token can be found on the control-plane node:

```bash
sudo cat /var/lib/rancher/k3s/server/node-token
```

Use it on each worker node:

```bash
curl -sfL https://get.k3s.io | \
K3S_URL=https://10.10.10.24:6443 \
K3S_TOKEN='YOUR_TOKEN_HERE' \
sh -s - agent \
  --node-name vanadium \
  --node-ip 10.10.10.21
```

This example joins `vanadium`.

Remember to adjust both the node name and IP address accordingly.

Run the command directly on the target machine.

Once all nodes have joined, verify the cluster:

```bash
kubectl get nodes -o wide
```

Every node should eventually report:

```text
Ready
```

Simple?

Simple.

A little time-consuming, perhaps, but there are no lengthy computations involved.

Only now is it safe to reach for your favorite book.
