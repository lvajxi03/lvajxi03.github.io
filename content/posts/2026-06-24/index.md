---
title: "2026-06-24"
description: "k3s: Ingress"
date: 2026-06-24T11:15:00+02:00
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
# k3s: Ingress

What exactly is an Ingress?

In the Kubernetes ecosystem, an **Ingress** is an API object that manages external access to applications running inside a cluster. It acts as an intelligent HTTP and HTTPS router, eliminating the need to create a separate load balancer for every service.

## How Does Ingress Work?

Traditional service exposure methods, such as `NodePort` or `LoadBalancer`, typically require assigning an external IP address to each service. This quickly becomes tedious and difficult to manage as the cluster grows.

Ingress centralizes external access and operates at Layer 7 (the application layer) of the OSI model:

* **Single entry point** — only one public IP address needs to be exposed to the outside world.
* **Rule-based routing** — traffic is forwarded to the appropriate service based on the requested URL path (for example, `my.domain/something`) or hostname (for example, `something.my.domain`).
* **SSL/TLS termination** — HTTPS certificates can be handled centrally, removing that responsibility from individual applications.

## Ingress vs. Ingress Controller

This distinction is important.

An Ingress object itself is merely a collection of routing rules.

For those rules to have any effect, the cluster must run an **Ingress Controller** — a reverse proxy responsible for processing incoming traffic and applying the rules defined in Ingress manifests.

Popular examples include:

* NGINX Ingress Controller
* Traefik
* HAProxy Ingress

This guide uses the **NGINX Ingress Controller**.

# Domain Names

Since the cluster will eventually host various web applications, it makes sense to identify services by DNS names rather than IP addresses.

The examples below use:

```text id="g7c9ux"
.lab.local
```

as the local domain.

This allows services to be accessed using names such as:

```text id="tvtqmk"
grafana.lab.local
argocd.lab.local
whoami.lab.local
```

which is considerably more pleasant than remembering port numbers.

# Certificates

Sooner or later, running everything without TLS becomes annoying.

For a home lab environment, generating your own Certificate Authority and issuing certificates locally is perfectly acceptable.

## Root Certificate Authority

Create a dedicated CA:

```bash id="0lbh0e"
mkdir -p certs
cd certs

openssl genrsa -out lab-ca.key 4096

openssl req -x509 -new -nodes \
  -key lab-ca.key \
  -sha256 \
  -days 3650 \
  -out lab-ca.crt \
  -subj "/CN=Lab Local CA"
```

## Wildcard Certificate

A wildcard certificate can secure every host under:

```text id="igmsib"
*.lab.local
```

Create the configuration file `lab-local.cnf`:

```text id="pt15ej"
[req]
default_bits = 2048
prompt = no
default_md = sha256
distinguished_name = dn
req_extensions = req_ext

[dn]
CN = *.lab.local

[req_ext]
subjectAltName = @alt_names

[alt_names]
DNS.1 = *.lab.local
DNS.2 = lab.local
```

Generate the key, request, and certificate:

```bash id="z6f8ju"
openssl genrsa -out lab-local.key 2048

openssl req -new \
  -key lab-local.key \
  -out lab-local.csr \
  -config lab-local.cnf

openssl x509 -req \
  -in lab-local.csr \
  -CA lab-ca.crt \
  -CAkey lab-ca.key \
  -CAcreateserial \
  -out lab-local.crt \
  -days 825 \
  -sha256 \
  -extensions req_ext \
  -extfile lab-local.cnf
```

# Namespace for the Ingress Controller

The ingress controller will live in its own namespace.

Create `ingress-namespace.yaml`:

```yaml id="x22dtm"
apiVersion: v1
kind: Namespace
metadata:
  name: ingress-nginx
```

Apply it:

```bash id="2afbrd"
kubectl apply -f ingress-namespace.yaml
```

# TLS Secret

The certificate must be stored inside Kubernetes as a TLS secret:

```bash id="shd7k8"
kubectl -n ingress-nginx create secret tls lab-local-tls \
  --cert=lab-local.crt \
  --key=lab-local.key
```

The secret is named:

```text id="qejv4z"
lab-local-tls
```

and can be referenced as:

```text id="4rwg0q"
ingress-nginx/lab-local-tls
```

throughout the cluster configuration.

# Installing the CA Certificate on Your Workstation

The root CA must also be trusted by your workstation:

```bash id="y8l6g8"
sudo cp lab-ca.crt /usr/local/share/ca-certificates/lab-ca.crt
sudo update-ca-certificates
```

Firefox users should additionally import the CA certificate manually through the browser settings.

# Deploying the Ingress Controller

A complete ingress controller deployment requires several Kubernetes resources:

* ServiceAccount
* IngressClass
* ConfigMap
* ClusterRole
* ClusterRoleBinding
* Deployment

The container image used in this setup is:

```text id="vwtkxu"
registry.k8s.io/ingress-nginx/controller:v1.14.0
```

which provides the familiar NGINX reverse proxy and load balancer functionality.

The deployment manifest is shown below.

```yaml id="9phd7v"
# ingress.yaml
# (manifest unchanged from the original article)
```

## Noteworthy Configuration

The ingress controller is pinned to the control-plane node:

```yaml id="kq6g9j"
nodeSelector:
  kubernetes.io/hostname: iridium
```

This is perfectly acceptable for a small home lab.

The controller also uses the wildcard certificate by default:

```yaml id="5jqhgm"
--default-ssl-certificate=ingress-nginx/lab-local-tls
```

which means every ingress can immediately benefit from TLS.

# Verifying the Installation

Start with the usual checks:

```bash id="my0z0j"
kubectl get pods -n ingress-nginx -o wide
kubectl get rs -n ingress-nginx -o wide
```

Next, add a test host entry:

```text id="zbx6ew"
10.10.10.24 test.lab.local
```

to your workstation's `/etc/hosts`.

Then run:

```bash id="5b6o9n"
curl http://test.lab.local
```

Expected result:

```html id="fhl3zq"
<html>
<head><title>404 Not Found</title></head>
<body>
<center><h1>404 Not Found</h1></center>
<hr><center>nginx</center>
</body>
</html>
```

Surprisingly, this is good news.

The DNS lookup works.

The ingress controller works.

NGINX works.

There simply isn't any application behind the hostname yet.

The infrastructure is doing exactly what it should.

## TLS Verification

Next, test HTTPS:

```bash id="9p72m9"
curl -v https://test.lab.local
```

The interesting part is not the final HTTP response, but the TLS negotiation itself.

Look for output similar to:

```text id="mjlwmh"
SSL connection using TLSv1.3
...
subject: CN=*.lab.local
...
issuer: CN=Lab Local CA
...
subjectAltName: "test.lab.local" matches cert's "*.lab.local"
...
SSL certificate verified via OpenSSL
```

If these messages appear, the certificate chain is functioning correctly and the wildcard certificate is being served as expected.

Finally, open the URL in Firefox and confirm that the browser displays the expected lock icon and certificate information.

# Conclusion

At this point:

* DNS works,
* the ingress controller is running,
* TLS works,
* wildcard certificates work,
* external traffic reaches the cluster.

No actual applications are exposed yet, but the foundation is now in place.

The cluster is finally ready to host something more interesting than infrastructure components.

Which, as every Kubernetes administrator eventually discovers, is usually the easy part.
