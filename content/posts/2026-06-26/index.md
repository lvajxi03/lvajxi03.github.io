---
title: "2026-06-26"
description: "Registry"
date: 2026-06-25T11:15:00+02:00
math: false
license: "CC BY-NC-SA 4.0"
hidden: false
comments: true
draft: false
tags:
    - Software
    - Kubernetes
categories:
    - Kubernetes
---
# Why Do We Need a Registry?

Sooner or later, every Kubernetes cluster starts running applications that were built locally.

At first, using Docker Hub seems perfectly reasonable.

Then the first custom application appears.

Then another.

Eventually every deployment references images that exist only on your workstation.

Clearly, this is not a sustainable solution.

A private container registry solves this problem by providing a permanent home for your own images. Since the home lab already has plenty of storage available, hosting the registry locally is both the simplest and the fastest option.

# PersistentVolume and PersistentVolumeClaim

One of the original design goals of Kubernetes was that Pods should be ephemeral.

Applications come and go.

Containers are recreated.

Nodes may disappear.

Unfortunately, databases tend to have rather strong opinions about losing their contents.

To bridge the gap between ephemeral workloads and persistent storage, Kubernetes introduced two API objects:

* **PersistentVolume (PV)**
* **PersistentVolumeClaim (PVC)**

## PersistentVolume

A **PersistentVolume** represents an actual storage resource available inside the cluster.

Unlike Pods, a PersistentVolume exists independently of any application using it. Deleting a Pod does not automatically remove the underlying storage.

Think of it as the physical disk—or, more precisely, a piece of storage that Kubernetes knows how to manage.

## PersistentVolumeClaim

A **PersistentVolumeClaim** is exactly what its name suggests: a request for storage.

Rather than asking for a specific disk, an application simply requests:

* a certain amount of space,
* an access mode,
* and, optionally, a storage class.

The Pod never mounts a PersistentVolume directly. Instead, it references a PVC and lets Kubernetes handle the rest.

## Binding

Whenever a PersistentVolumeClaim is created, Kubernetes searches for a suitable PersistentVolume that satisfies the requested parameters.

If one is found, the claim becomes permanently bound to that volume.

From the application's point of view, none of this complexity exists.

The Pod simply mounts the PVC as a directory inside the container.

## Access Modes

Access modes describe **how Kubernetes nodes are allowed to mount a volume**.

Four access modes currently exist:

* **ReadWriteOnce (RWO)** — the volume may be mounted for reading and writing by only **one node** at a time. Multiple Pods may still use it, provided they all run on that node.

* **ReadWriteMany (RWX)** — the volume may be mounted simultaneously by multiple nodes.

* **ReadOnlyMany (ROX)** — multiple nodes may mount the volume, but only for reading.

* **ReadWriteOncePod (RWOP)** — introduced more recently, this mode guarantees that exactly one Pod in the entire cluster has read-write access to the volume.

> **Note:** the underlying storage backend must actually support the requested access mode. Kubernetes cannot magically turn a local SSD into a distributed filesystem.

# Where Should the Data Live?

By default, **k3s** ships with the **Local Path Provisioner**, which stores dynamically created PersistentVolumes under:

```text
/var/lib/rancher/k3s/storage
```

Personally, I prefer keeping persistent application data somewhere more obvious, such as:

```text
/srv/storage
```

To do that, create the directory on every node:

```bash
sudo mkdir -p /srv/storage
sudo chmod 777 /srv/storage
```

The permissions are intentionally generous. This is a home lab, not Fort Knox.

Next, edit the `local-path-config` ConfigMap:

```bash
kubectl -n kube-system edit configmap local-path-config
```

The interesting part looks like this:

```yaml
apiVersion: v1
data:
  config.json: |-
    {
      "nodePathMap":[
      {
        "node":"DEFAULT_PATH_FOR_NON_LISTED_NODES",
        "paths":["/var/lib/rancher/k3s/storage"]
      }
      ]
    }
```

Simply replace:

```json
"paths":["/var/lib/rancher/k3s/storage"]
```

with:

```json
"paths":["/srv/storage"]
```

Newly created PersistentVolumes will now live there instead.

# Deploying the Registry

The registry will be available as:

```text
registry.lab.local
```

Since there is no local DNS server yet, every machine—including your workstation—should receive the following `/etc/hosts` entry:

```text
10.10.10.24 registry.lab.local
```

The registry should preferably run on the most capable machine in the cluster.

In my case, that is `iridium`.

Assign it an appropriate label:

```bash
kubectl label node iridium registry=true
```

The Deployment manifest below uses that label via a `nodeSelector`.

At first glance, a registry seems like a simple application.

Then you remember it needs persistent storage.

And a Service.

And an Ingress.

And perhaps its own namespace.

And a ConfigMap.

So, in the end, it turns out to require exactly six Kubernetes objects:

* Namespace
* PersistentVolumeClaim
* ConfigMap
* Deployment
* Service
* Ingress

Their complete definitions are shown below.

*(YAML manifest unchanged.)*

Save everything as:

```text
registry.yaml
```

and deploy it using:

```bash
kubectl apply -f registry.yaml
```

## Interesting Bits

The PVC requests persistent storage:

```yaml
accessModes:
  - ReadWriteOnce
```

The registry is pinned to the node labelled earlier:

```yaml
nodeSelector:
  registry: "true"
```

The ingress disables request size limits:

```yaml
nginx.ingress.kubernetes.io/proxy-body-size: "0"
```

Without this annotation, pushing sufficiently large images eventually results in sadness, disappointment, and extensive arm waving.

The registry configuration itself is stored in a ConfigMap rather than a long list of environment variables.

Since the registry already expects a YAML configuration file, using a ConfigMap keeps the deployment considerably cleaner.

> This setup stores all registry data on the local filesystem of a single node. That is perfectly adequate for a home lab. Production environments usually rely on distributed storage instead.

# Running Without TLS

If the registry temporarily runs over plain HTTP, Docker must be told to trust it.

Create:

```text
/etc/docker/daemon.json
```

containing:

```json
{
  "insecure-registries": ["registry.lab.local"]
}
```

Likewise, every k3s node should receive:

```text
/etc/rancher/k3s/registries.yaml
```

containing:

```yaml
mirrors:
  "registry.lab.local":
    endpoint:
      - "http://registry.lab.local"
```

Finally, restart k3s:

```bash
sudo systemctl restart k3s || sudo systemctl restart k3s-agent
```

Don't forget to update the ConfigMap as well if you're switching from HTTPS to HTTP.

# Verification

The usual suspects come first:

```bash
kubectl get pods -n registry
kubectl get pvc -n registry
kubectl get ingress -n registry
kubectl get svc -n registry
kubectl describe ingress registry -n registry
kubectl get configmap -n registry
```

If the PVC reports:

```text
STATUS: Bound
```

everything is proceeding nicely:

* the PersistentVolume exists,
* the claim has been bound,
* the Pod is successfully using it.

# Smoke Test

Let's put the registry to work.

```bash
docker pull php:8.4.22-zts-trixie

docker tag php:8.4.22-zts-trixie \
    registry.lab.local/php:8.4.22

docker push registry.lab.local/php:8.4.22
```

To make sure the image really came from the registry rather than your local cache, remove it first:

```bash
docker image rm registry.lab.local/php:8.4.22

docker pull registry.lab.local/php:8.4.22
```

Finally, query the registry directly:

```bash
curl https://registry.lab.local/v2/php/tags/list
```

which should return:

```json
{"name":"php","tags":["8.4.22"]}
```

Or list every repository:

```bash
curl -s https://registry.lab.local/v2/_catalog | jq
```

Congratulations.

Your registry is officially alive.

# Registry UI

A registry works perfectly well without a graphical interface.

Humans, however, tend to appreciate one.

The excellent **Docker Registry UI** provides browsing, searching, and deleting images from a web browser.

As before, add:

```text
10.10.10.24 registry-ui.lab.local
```

to every `/etc/hosts`.

Deploy the accompanying Deployment, Service, and Ingress manifests (shown below), save them as:

```text
registry-ui.yaml
```

and apply them:

```bash
kubectl apply -f registry-ui.yaml
```

The interface is intentionally simple.

It won't win any design awards, but it makes exploring your registry significantly more pleasant than remembering API endpoints.

# Wrapping Up

At this point, the cluster has become largely self-sufficient.

Applications can now be built locally, pushed into the private registry, and deployed without relying on Docker Hub.

It may not look particularly impressive yet, but from now on every new service added to the cluster can remain entirely within the home lab.

Which, after all, was the idea from the very beginning.
