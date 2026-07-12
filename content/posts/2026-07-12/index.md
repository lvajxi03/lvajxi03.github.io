---
title: "Jenkins, Part Two: Disposable Agents and Builder Images"
description: "Jenkins, Part II: images, ephemeral agents, and practical use cases"
date: 2026-07-12T17:20:00+02:00
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

# Once again: why all these containers?

The Kubernetes approach — short-lived, disposable things — works beautifully for Jenkins and similar CI/CD systems.

Here is a list of tasks to execute in a place described by a specification, not by a physical location. Whether that place existed five minutes ago or will disappear five minutes from now does not matter all that much.

Look at how the approach to running CI/CD jobs has evolved over the last twenty years:

* run the job in a blocking shell script,
* run the job directly on the Jenkins master,
* run the job on a dedicated Jenkins node,
* run the job on one of several dedicated Jenkins nodes.

And now?

* run the job inside an ephemeral Pod, created just before the work starts and removed when the work is done.

Hard not to admire that, is it?

And there is more to admire. Each container can provide a different set of tools. Instead of installing everything manually on physical machines, you can point Jenkins at a ready-made image, built by someone else — and if you do not like that one, you can use another.

All right, that is enough admiration for now.

# What do we already have?

Quite a lot:

* **Jenkins** running under Kubernetes,
* a working **container registry**,
* a **PVC** for Jenkins data,
* a **Kubernetes cloud** definition in Jenkins,
* two already built **images** that we are about to use.

# What else do we need?

Eventually, Jenkins will run jobs such as:

* building Python applications,
* building Go applications,
* building Hugo sites.

So we need three images.

# Python builder

The Python image can be built from the following `Dockerfile`:

```dockerfile id="xo7x23"
FROM registry.lab.local/base-builder:0.1

USER root

RUN apt-get update && apt-get install -y --no-install-recommends \
    python3 \
    python3-pip \
    python3-venv \
    python3-build \
    && rm -rf /var/lib/apt/lists/*

RUN python3 -m pip install --break-system-packages \
    build \
    twine \
    wheel \
    setuptools \
    pytest \
    ruff

USER builder
WORKDIR /workspace

CMD ["sleep", "infinity"]
```

Wait a moment. Do not reach for `docker build` yet.

Actually, do not reach for it at all.

Put this `Dockerfile` into a source repository, preferably in a dedicated subdirectory. Later it will be much easier to tell what kind of creatures are being produced from it.

> We are about to enable a tiny bit of magic. If you are not ready, perhaps take a short walk.
>
> Jenkins can wait. That is, after all, usually its main job.

Create a `Jenkinsfile` with the following contents:

```groovy id="ozj9af"
pipeline {
    agent {
        kubernetes {
            yaml '''
apiVersion: v1
kind: Pod
spec:
  hostAliases:
    - ip: "10.10.10.24"
      hostnames:
        - "registry.lab.local"
  containers:
    - name: base
      image: registry.lab.local/base-builder:0.1
      command: ["sleep"]
      args: ["infinity"]
    - name: kaniko
      image: registry.lab.local/kaniko-builder:latest
      command: ["sleep"]
      args: ["infinity"]
      volumeMounts:
        - name: docker-config
          mountPath: /kaniko/.docker
  volumes:
    - name: docker-config
      configMap:
        name: kaniko-docker-config
'''
        }
    }

    environment {
        IMAGE_NAME = 'registry.lab.local/python-builder'
        IMAGE_TAG = '3.11.1'
    }

    stages {
        stage('Prepare') {
            steps {
                container('base') {
                    script {
                        env.GIT_SHA = sh(
                            script: 'git rev-parse --short HEAD',
                            returnStdout: true
                        ).trim()

                        currentBuild.displayName = "#${env.BUILD_NUMBER} python-3"
                        currentBuild.description = "python-builder:3, commit ${env.GIT_SHA}"
                    }

                    sh '''
                        echo "IMAGE=${IMAGE_NAME}:${IMAGE_TAG}"
                        ls -la
                    '''
                }
            }
        }

        stage('Build and push image') {
            steps {
                container('kaniko') {
                    sh '''
                        /kaniko/executor \
                          --context "${WORKSPACE}" \
                          --dockerfile "${WORKSPACE}/python-builder/Dockerfile" \
                          --destination "${IMAGE_NAME}:${IMAGE_TAG}" \
                          --insecure \
                          --skip-tls-verify \
                          --insecure-registry registry.lab.local
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "Built and pushed ${IMAGE_NAME}:${IMAGE_TAG}"
        }
    }
}
```

Commit it to the same repository, in the same subdirectory as the `Dockerfile`.

The cluster needs one additional **ConfigMap**:

```yaml id="8sf2o7"
apiVersion: v1
kind: ConfigMap
metadata:
  name: kaniko-docker-config
  namespace: jenkins
data:
  config.json: |
    {
      "auths": {}
    }
```

The `Jenkinsfile` refers to this ConfigMap in the `agent` section.

Now create a new **Multibranch Pipeline** job. Give it a sensible name, for example `python-builder`, and configure it:

* enter the repository URL,
* if credentials are required, select them, or add them first,
* under **Build configuration** → **Mode**, make sure **by Jenkinsfile** is selected,
* in **Script Path**, enter the path to the file, for example `python-builder/Jenkinsfile`.

Save the job and wait a moment.

Jenkins will start scanning branches in your repository, looking for the specified `Jenkinsfile` in each of them. If it finds one, it will try to run the job for that branch.

Once the **Stage View** starts rendering, you can observe what is happening in the `jenkins` namespace:

```bash id="dionr0"
kubectl get pods -n jenkins -w
```

A new Pod will appear, do its job, and leave, as in the example below:

```text id="zjpqkv"
NAME                                           READY   STATUS              RESTARTS       AGE
jenkins-0                                      1/1     Running             12 (57m ago)   19d
python-builder-main-1-30xvv-l1cp7-vcw9m       0/3     Pending             0              0s
python-builder-main-1-30xvv-l1cp7-vcw9m       0/3     ContainerCreating   0              0s
python-builder-main-1-30xvv-l1cp7-vcw9m       3/3     Running             0              1s
python-builder-main-1-30xvv-l1cp7-vcw9m       3/3     Terminating         0              63s
python-builder-main-1-30xvv-l1cp7-vcw9m       0/3     Error               0              93s
```

The job completed successfully and the `python-builder` image landed in the registry.

You can verify that either through `registry-ui` or directly:

```bash id="ugqs7n"
curl http://registry.lab.local/v2/python-builder/tags/list
```

The response should look like this:

```json id="3rqvyc"
{"name":"python-builder","tags":["3.11.1"]}
```

The image is now ready to use.

# What just happened?

The contents of the `Jenkinsfile` definitely deserve an explanation, so let us start from the beginning.

## `agent`

The `agent` section is without a doubt the most interesting part of this file.

Besides the type name, `kubernetes`, it contains a full Kubernetes Pod definition. In the specification we can see that:

* `hostAliases` are used, because we do not have proper DNS here and still need to smuggle the registry address into the Pod,
* all important containers are listed:

  * `base`, which provides Git and basic shell tools,
  * `kaniko`, which provides... well, kaniko.

Keeping the names consistent is important.

If you name the containers something like `the_one_with_git` and `the_one_with_kaniko`, you will have to use exactly those names later in the actual pipeline steps:

```groovy id="9n8v86"
stages {
    stage('Prepare') {
        steps {
            container('the_one_with_git') {
```

In short: Jenkins creates an agent described in the `agent` section, and pipeline steps are executed inside specific containers from that Pod — the ones that provide the required tools.

This is the important part.

We are not installing build tools inside the main Jenkins server. The server only orchestrates work. The tools live in disposable containers, created only for the job that needs them.

That keeps the Jenkins controller cleaner, makes builds easier to reproduce, and avoids the traditional CI server problem where one machine slowly turns into a museum of forgotten compilers, package managers, SDKs, symlinks and emotional damage.

## `kaniko`

Remember that `kaniko` is not Docker.

Its job is simply to build a container image and push it to a registry.

The important parameters are:

* `--context` — the working directory, usually the job workspace,
* `--dockerfile` — the location of the `Dockerfile`, in this case inside a subdirectory,
* `--destination` — the registry location, image name and tag to push after the build.

Simple? Simple.

"But wait!" you might shout. "What about that mysterious `kaniko-docker-config` ConfigMap?"

Here it is.

`kaniko` likes to look inside `/kaniko/.docker`, mainly for a `config.json` file containing credentials, if they are required.

Our registry does not require authentication, but we still mount a ConfigMap there and provide an empty `auths` object:

```json id="1hiwcy"
{
  "auths": {}
}
```

This allows kaniko to try pushing the image anonymously right away.

# Go builder

Not enough examples? Fine. Let us create a `go-builder`.

The `Dockerfile` could look like this:

```dockerfile id="usgtsu"
FROM golang:1.26-bookworm

RUN apt-get update && apt-get install -y --no-install-recommends \
    git \
    openssh-client \
    ca-certificates \
    curl \
    wget \
    jq \
    make \
    rsync \
    && rm -rf /var/lib/apt/lists/*

RUN update-ca-certificates

USER root

WORKDIR /workspace

RUN go version

CMD ["sleep", "infinity"]
```

And the `Jenkinsfile` could look like this:

```groovy id="kfnob9"
pipeline {
    agent {
        kubernetes {
            yaml '''
apiVersion: v1
kind: Pod
spec:
  hostAliases:
    - ip: "10.10.10.24"
      hostnames:
        - "registry.lab.local"
  containers:
    - name: base
      image: registry.lab.local/base-builder:0.1
      command: ["sleep"]
      args: ["infinity"]
    - name: kaniko
      image: registry.lab.local/kaniko-builder:latest
      command: ["sleep"]
      args: ["infinity"]
      volumeMounts:
        - name: docker-config
          mountPath: /kaniko/.docker
  volumes:
    - name: docker-config
      configMap:
        name: kaniko-docker-config
'''
        }
    }

    environment {
        IMAGE_NAME = 'registry.lab.local/go-builder'
        IMAGE_TAG = '1.26.1'
    }

    stages {
        stage('Prepare') {
            steps {
                container('base') {
                    script {
                        env.GIT_SHA = sh(
                            script: 'git rev-parse --short HEAD',
                            returnStdout: true
                        ).trim()

                        currentBuild.displayName = "#${env.BUILD_NUMBER} go-${env.IMAGE_TAG}"
                        currentBuild.description = "go-builder:${env.IMAGE_TAG}, commit ${env.GIT_SHA}"
                    }

                    sh '''
                        echo "GO_VERSION=${IMAGE_TAG}"
                        echo "IMAGE=${IMAGE_NAME}:${IMAGE_TAG}"
                        ls -la
                    '''
                }
            }
        }

        stage('Build and push image') {
            steps {
                container('kaniko') {
                    sh '''
                        /kaniko/executor \
                          --context "${WORKSPACE}" \
                          --dockerfile "${WORKSPACE}/go-builder/Dockerfile" \
                          --destination "${IMAGE_NAME}:${IMAGE_TAG}" \
                          --insecure \
                          --skip-tls-verify \
                          --insecure-registry registry.lab.local
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "Built and pushed ${IMAGE_NAME}:${IMAGE_TAG}"
        }
    }
}
```

Yes, you do not need to rub your eyes in disbelief.

This `Jenkinsfile` looks almost identical to the previous one. The agent definition is the same, the `kaniko` invocation is the same, and the base image usage is the same. In practice, only names, version numbers and file paths change.

Simple? Simple.

# Example use case

If you have a Go application lying around, you can use a `Jenkinsfile` like this one.

In this example, the test application is called `rme`:

```groovy id="gxszsp"
pipeline {
    agent {
        kubernetes {
            yaml '''
apiVersion: v1
kind: Pod
spec:
  hostAliases:
    - ip: "10.10.10.24"
      hostnames:
        - "registry.lab.local"
  containers:
    - name: rme-go-builder
      image: registry.lab.local/go-builder:1.26.1
      command: ["sleep"]
      args: ["infinity"]
    - name: rme-kaniko
      image: registry.lab.local/kaniko-builder:latest
      command: ["sleep"]
      args: ["infinity"]
      volumeMounts:
        - name: docker-config
          mountPath: /kaniko/.docker
  volumes:
    - name: docker-config
      configMap:
        name: kaniko-docker-config
'''
        }
    }

    environment {
        IMAGE_NAME = 'registry.lab.local/rme'
    }

    stages {
        stage('Prepare') {
            steps {
                container('rme-go-builder') {
                    script {
                        env.VERSION = env.GIT_COMMIT.take(7)

                        currentBuild.displayName = "#${env.BUILD_NUMBER} ${env.VERSION}"
                        currentBuild.description = "${env.IMAGE_NAME}:${env.VERSION}"
                    }
                }
            }
        }

        stage('Build binary') {
            steps {
                container('rme-go-builder') {
                    sh '''
                        git config --global --add safe.directory "$WORKSPACE"
                        CGO_ENABLED=0 GOOS=linux go build -o rme .
                    '''
                }
            }
        }

        stage('Build and push image') {
            steps {
                container('rme-kaniko') {
                    sh '''
                        /kaniko/executor \
                          --context "${WORKSPACE}" \
                          --dockerfile "${WORKSPACE}/Dockerfile" \
                          --destination "${IMAGE_NAME}:${VERSION}" \
                          --destination "${IMAGE_NAME}:latest" \
                          --insecure \
                          --skip-tls-verify \
                          --insecure-registry registry.lab.local
                    '''
                }
            }
        }
    }
}
```

The application `Dockerfile` can be very small:

```dockerfile id="p901cs"
FROM scratch

COPY rme /rme

EXPOSE 9101

ENTRYPOINT ["/rme"]
```

This is a more realistic example.

It uses a dedicated builder image, `go-builder`, to compile the Go application, and `kaniko` only for building and pushing the final container image.

A few details are worth noticing, because the devil usually lives exactly there:

* container names matter, because they are later used in `container(...)` blocks,
* the second `--destination` in the `kaniko` invocation:

```bash id="7w14hb"
--destination "${IMAGE_NAME}:latest" \
```

is intentional.

The `latest` tag will not magically appear in our registry by itself. If we want the most recently pushed image to also be available as `latest`, we need to tag and push it that way.

# Summary and notes

As you can see, there is less and less that we strictly **must** do inside the cluster, and more and more that we **can** do in a controlled and repeatable way.

Versioned configuration, job history, artifact history, example applications, builds running inside containers — quite a lot is happening here.

Why use a **Multibranch Pipeline**?

Good question, but the answers are fairly obvious:

* we do not have to clone the repository manually or hard-code credentials in the `Jenkinsfile`,
* test branches do not require separate Jenkins jobs,
* Jenkins detects new branches automatically,
* deleted branches can disappear from Jenkins automatically as well,
* each branch can have its own version of the `Jenkinsfile`,
* pull requests are handled in a much cleaner way.

---

And now a suspicious thought appears.

What if we put all those YAML files under version control and let some clever mechanism take care of them automatically, instead of calling `kubectl` by hand every few minutes?

Could such a thing exist?

Who knows.
