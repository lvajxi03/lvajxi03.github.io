---
title: "Jenkins in Kubernetes: Bring a Bucket"
description: "Jenkins"
date: 2026-07-11T11:10:00+02:00
math: false
license: "CC BY-NC-SA 4.0"
hidden: false
comments: true
draft: false
tags:
    - Software
    - Kubernetes
    - Jenkins
categories:
    - Kubernetes
series:
    - Kubernetes from Zero to Headaches
---

> **Warning:** working with Jenkins requires a lot of determination. It is also wise to prepare a bucket in advance — for blood, sweat, tears, and all the failed pipelines.
>
> There. I warned you.

# Why Jenkins?

Jenkins has accumulated enough myths and urban legends to deserve several dedicated episodes of a good television show about mysterious phenomena and conspiracy theories.

For now, let us focus on a few practical facts:

* Jenkins is a true Swiss Army knife in the CI/CD world.
* Jenkins is worth knowing, because it may be lurking just around the corner in the organization you are about to join.
* Jenkins is easy to run both locally and centrally.
* Thanks to its many plugins, Jenkins can probably be used for almost anything.
* You do not have to spend your life clicking around the UI. Jenkins Pipelines do their job.
* It is worth getting familiar with Jenkins if only to form your own opinion about what you actually need from a CI/CD ecosystem.

# A pinch of history

Jenkins originally started as a tool for building Java code, wrapped in a convenient web service available to everyone.

At first it was called **Hudson**, and perhaps it would still be called that today, but Sun Microsystems — where Hudson was created — was acquired by the large and not particularly cuddly company known for large and not particularly agile databases.

Then came a dispute over the name and project ownership. A fork appeared, someone proposed a new name, and off it went.

If this reminds you of MariaDB, congratulations: your pattern recognition module is working correctly.

# Jenkins in a Kubernetes cluster

We will use Jenkins not only to build containers, but also to build regular applications and run ordinary tasks using purpose-built containers.

## Requirements

Jenkins stores data on disk, so we will need a PersistentVolumeClaim.

From the cluster's point of view, the main Jenkins server will run as a StatefulSet. An Ingress controller will provide HTTP(S) access through the address `jenkins.lab.local`.

As usual in this lab, add the following line to `/etc/hosts` on every machine that needs to access Jenkins, including your workstation:

```text
10.10.10.24 jenkins.lab.local
```

## The image

You can use an existing Jenkins image, for example the latest LTS image.

I do not recommend doing that here.

The bucket for blood, sweat and tears would need to be emptied far too often, with very little benefit. It is better to commit a custom image yourself — or in pairs, because at some point the basement nerds need someone to hold the light bulb.

## Dockerfile

The `Dockerfile` looks like this:

```dockerfile
FROM jenkins/jenkins:lts-jdk21 AS jenkins-tools

FROM eclipse-temurin:21-jre-jammy

ARG JENKINS_VERSION=2.555.3

USER root

RUN apt-get update && apt-get install -y --no-install-recommends \
    curl \
    git \
    openssh-client \
    ca-certificates \
    tini \
    fontconfig \
    jq \
    unzip \
    zip \
    && rm -rf /var/lib/apt/lists/*

RUN useradd -m -d /var/jenkins_home -u 1000 jenkins

RUN mkdir -p /usr/share/jenkins /var/jenkins_home /usr/share/jenkins/ref/plugins

RUN curl -fsSL \
    -o /usr/share/jenkins/jenkins.war \
    "https://get.jenkins.io/war-stable/${JENKINS_VERSION}/jenkins.war"

COPY --from=jenkins-tools /usr/bin/jenkins-plugin-cli /usr/bin/jenkins-plugin-cli
COPY --from=jenkins-tools /opt/jenkins-plugin-manager.jar /opt/jenkins-plugin-manager.jar

COPY plugins.txt /tmp/plugins.txt

RUN JENKINS_WAR=/usr/share/jenkins/jenkins.war \
    JENKINS_HOME=/usr/share/jenkins/ref \
    jenkins-plugin-cli \
      --war /usr/share/jenkins/jenkins.war \
      --plugin-file /tmp/plugins.txt

RUN chown -R jenkins:jenkins /usr/share/jenkins /var/jenkins_home

USER jenkins

ENV JENKINS_HOME=/var/jenkins_home

VOLUME /var/jenkins_home

EXPOSE 8080 50000

ENTRYPOINT ["/usr/bin/tini", "--"]
CMD ["java", "-Djenkins.install.runSetupWizard=false", "-jar", "/usr/share/jenkins/jenkins.war", "--httpPort=8080"]
```

You can put plugins in `plugins.txt` and install them directly into the image.

You can also install them later, because Jenkins will have persistent storage. Whether this is a convenience or the beginning of a future archaeology project depends on your level of optimism.

# Building the image

The image is built for the `arm64` architecture:

```bash
docker buildx build \
  --platform linux/arm64 \
  -t registry.lab.local/jenkins-lab:0.0.1 \
  --push .
```

The most likely result at this point is something like this:

```text
> [stage-1  2/11] RUN apt-get update && apt-get install -y --no-install-recommends     curl     git     openssh-client     ca-certificates     tini     fontconfig     jq     unzip     zip     && rm -rf /var/lib/apt/lists/*:
0.126 exec /bin/sh: exec format error
```

Which means we need to help Docker understand how to run `arm64` binaries:

```bash
docker run --privileged --rm tonistiigi/binfmt --install arm64
```

A successful installation should produce something like this:

```text
Installing: arm64 OK
{
  "supported": [
    "linux/amd64",
    "linux/amd64/v2",
    "linux/amd64/v3",
    "linux/arm64",
    "linux/386"
  ],
  "emulators": [
    "python3.14",
    "qemu-aarch64"
  ]
}
```

Now we can build the image again:

```bash
docker buildx build \
  --platform linux/arm64 \
  -t registry.lab.local/jenkins-lab:0.0.1 \
  --push .
```

# Cluster objects

For Jenkins to work properly, we need a few Kubernetes objects.

## Namespace

The simplest way to create the namespace is:

```bash
kubectl create namespace jenkins
```

If you keep your configuration under version control — and you should, because future you has enough problems already — use a manifest instead:

```yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: jenkins
```

## PersistentVolumeClaim

The PVC definition looks like this:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: jenkins-home
  namespace: jenkins
spec:
  accessModes:
    - ReadWriteOnce

  resources:
    requests:
      storage: 20Gi

  storageClassName: local-path
```

## ServiceAccount and Role

Jenkins needs permission to create and manage agent Pods in its namespace.

The following definitions are enough for this lab:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins
  namespace: jenkins
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: jenkins-agent-manager
  namespace: jenkins
rules:
  - apiGroups: [""]
    resources: ["pods", "pods/exec", "pods/log", "persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "create", "delete", "patch", "update"]
  - apiGroups: [""]
    resources: ["secrets", "configmaps"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: jenkins-agent-manager
  namespace: jenkins
subjects:
  - kind: ServiceAccount
    name: jenkins
    namespace: jenkins
roleRef:
  kind: Role
  name: jenkins-agent-manager
  apiGroup: rbac.authorization.k8s.io
```

## Service and Ingress

Now we need a Service and an Ingress:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: jenkins
  namespace: jenkins
spec:
  selector:
    app: jenkins
  ports:
    - name: http
      port: 8080
      targetPort: http
    - name: agent
      port: 50000
      targetPort: agent
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: jenkins
  namespace: jenkins
spec:
  ingressClassName: nginx
  rules:
    - host: jenkins.lab.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: jenkins
                port:
                  name: http
```

## StatefulSet

Finally, the StatefulSet:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: jenkins
  namespace: jenkins
spec:
  serviceName: jenkins
  replicas: 1

  selector:
    matchLabels:
      app: jenkins

  template:
    metadata:
      labels:
        app: jenkins

    spec:
      serviceAccountName: jenkins
      nodeSelector:
        przeznaczenie: ci-cd

      volumes:
        - name: jenkins-home
          persistentVolumeClaim:
            claimName: jenkins-home

      containers:
        - name: jenkins
          image: registry.lab.local/jenkins-lab:0.0.1
          imagePullPolicy: Always

          ports:
            - name: http
              containerPort: 8080
            - name: agent
              containerPort: 50000

          env:
            - name: JENKINS_HOME
              value: /var/jenkins_home
            - name: JAVA_OPTS
              value: "-Djenkins.install.runSetupWizard=false"

          volumeMounts:
            - name: jenkins-home
              mountPath: /var/jenkins_home

          startupProbe:
            httpGet:
              path: /login
              port: http
            periodSeconds: 10
            failureThreshold: 60

          readinessProbe:
            httpGet:
              path: /login
              port: http
            periodSeconds: 10
            failureThreshold: 12

          livenessProbe:
            httpGet:
              path: /login
              port: http
            periodSeconds: 30
            failureThreshold: 10
```

> Save all of the above definitions as YAML files and apply them with `kubectl`.

# First start

Jenkins should start and greet us by asking for the initial token.

That token can be found in the depths of its home directory on the PVC. After that, it is the usual Jenkins ritual: administrator account, password, pending plugins, and all the other traditional rites of passage.

A freshly started Jenkins is a strange creature.

On one hand, it can be armed to the teeth with plugins. On the other hand, it is as naked as a jaybird. It does not have any build tools installed — but that is fine. We do not really need them.

The actual work will be done by purpose-built containers, created only when needed.

# Configuring the Kubernetes cloud

This is where the Jenkins **Cloud** configuration comes in.

Configure it like this:

* **Manage Jenkins** → **Clouds** → **Add a new cloud** → **Kubernetes**
* **Name:** `kubernetes`, or any other name you prefer
* **Kubernetes URL:** `https://kubernetes.default.svc`
* **Kubernetes namespace:** `jenkins`
* Enable **WebSocket**
* **Jenkins URL:** `http://jenkins.jenkins.svc.cluster.local:8080`

Save the configuration.

Now all that remains is to create a few useful images, wire them into Jenkins pipelines, and then use them, use them, and use them some more.

# Base builder

The first image will be... a base image for building other, better base images.

What does that mean?

The plan is for Jenkins to build many different applications, and also run jobs that may not build anything at all, but still need containers with specific tools inside.

There are two ways to approach this problem:

* create one large image with every possible tool inside,
* create many small, specialized images and use each one only when needed.

The second approach is easier to maintain.

The `Dockerfile` for the **base builder** looks roughly like this:

```dockerfile
FROM debian:bookworm-slim

RUN apt-get update && apt-get install -y --no-install-recommends \
    ca-certificates \
    curl \
    wget \
    git \
    openssh-client \
    bash \
    make \
    jq \
    yq \
    tar \
    gzip \
    xz-utils \
    unzip \
    zip \
    rsync \
    file \
    procps \
    iputils-ping \
    dnsutils \
    && rm -rf /var/lib/apt/lists/*

RUN useradd -m -u 1000 builder

USER builder
WORKDIR /workspace

CMD ["sleep", "infinity"]
```

Build it like this:

```bash
docker buildx build \
  --platform linux/arm64 \
  -t registry.lab.local/base-builder:0.1 \
  --push .
```

# Kaniko builder

The second image will contain **kaniko**, mainly for building container images.

> **Note:** this is a lab setup. Kaniko works well enough for our purposes here, but before using it in production, check the current state of the project and consider alternatives such as Buildah or Podman. The CI/CD ecosystem moves quickly, usually while pretending it is doing so in a controlled manner.

Yes, kaniko is simpler to use than Docker in this setup. It does not require root privileges and it does not need a Docker daemon.

The `Dockerfile` can look like this:

```dockerfile
FROM gcr.io/kaniko-project/executor:latest AS kaniko

FROM alpine:3.21

RUN apk add --no-cache \
    ca-certificates \
    bash \
    coreutils \
    curl \
    git

COPY --from=kaniko /kaniko /kaniko

ENV PATH="/kaniko:${PATH}"

WORKDIR /workspace

CMD ["sleep", "infinity"]
```

Build it like this:

```bash
docker buildx build \
  --platform linux/arm64 \
  -t registry.lab.local/kaniko-builder:0.1 \
  --push .
```

These two images need to be built manually.

All the other ones will be created later by Jenkins itself.

If you managed to get here in one piece, that is already quite good. In the next episode, we will learn how to make actual use of our newly assembled toys.

