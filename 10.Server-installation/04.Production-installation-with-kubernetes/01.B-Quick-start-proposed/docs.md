---
title: Demo Quick Start (Proposed)
taxonomy:
    category: docs
    label: tutorial
---

!! This page installs a single-node Mender Server for demo and evaluation
!! only. The Helm chart runs its own MongoDB, Redis and RustFS artifact storage,
!! and generates a self-signed TLS certificate. For production, follow the
!! [Kubernetes](../01.Kubernetes/docs.md), [Storage](../02.Storage/docs.md) and
!! [Mender Server](../03.Mender-server/docs.md) sections instead.

<!--AUTOVERSION: "requires the Mender Helm chart version % or later"/ignore -->
This page requires the Mender Helm chart version 8.6.0 or later.

## Prerequisites

* A Linux host with at least 4 vCPUs and 8 GB RAM, with `curl`, `openssl` and root access.
* A host name for the server that your devices and browser resolve to the host,
  through DNS or `/etc/hosts`. Port 443 must be reachable.
* At least 1024 inotify instances per user (`fs.inotify.max_user_instances`).
  Many distributions default to 128, and the Mender Server Pods use most of them.
  The next step raises the limit.

## Install K3s and Helm

!!! Review scripts before you pipe them to a shell.

```bash
echo "fs.inotify.max_user_instances = 1024" | sudo tee /etc/sysctl.d/90-mender-inotify.conf
sudo sysctl -p /etc/sysctl.d/90-mender-inotify.conf

curl -sfL https://get.k3s.io | sh -
mkdir -p ~/.kube
sudo install -m 600 -o "$(id -u)" -g "$(id -g)" /etc/rancher/k3s/k3s.yaml ~/.kube/config
export KUBECONFIG=~/.kube/config

curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

## Install the Mender Server

The values file is readable only by you, because it contains the MongoDB password.
Keep it for later upgrades: MongoDB stores its data on a persistent volume and
keeps the root password it was first started with.

[ui-tabs position="top-left" active="0" theme="default" ]
[ui-tab title="Open Source"]

Create the values file:

```bash
export MENDER_SERVER_DOMAIN="mender.example.com"
umask 077
cat >mender-values.yml <<EOF
global:
  url: "https://${MENDER_SERVER_DOMAIN}"

ingress:
  enabled: true
  ingressClassName: traefik
  path: /
  hosts:
    - ${MENDER_SERVER_DOMAIN}
  tls:
    - secretName: mender-ingress-tls
      hosts:
        - ${MENDER_SERVER_DOMAIN}
  selfSignedTLS:
    enabled: true

rustfs:
  enabled: true

mongodb:
  auth:
    rootPassword: "$(openssl rand -hex 24)"
  persistence:
    enabled: true

redis:
  enabled: true
EOF
```

Install the Mender Server:

```bash
helm repo add mender https://charts.mender.io
helm repo update
helm upgrade --install mender mender/mender --wait --timeout 15m -f mender-values.yml
```

Create the first user:

```bash
kubectl exec deploy/mender-useradm -- useradm create-user \
  --username "replace-with-your-email" --password "replace-with-your-password"
```

[/ui-tab]
[ui-tab title="Enterprise"]

!!!!! The Enterprise images are in the Mender Enterprise Container Registry.
!!!!! Email [contact@mender.io](mailto:contact@mender.io) for an evaluation account.

Create the Docker registry secret:

```bash
read -r -p "Registry username: " MENDER_REGISTRY_USERNAME
read -r -s -p "Registry password: " MENDER_REGISTRY_PASSWORD; echo

kubectl create secret docker-registry my-mender-pull-secret \
  --docker-server=registry.mender.io \
  --docker-username="${MENDER_REGISTRY_USERNAME}" \
  --docker-password="${MENDER_REGISTRY_PASSWORD}"
unset MENDER_REGISTRY_PASSWORD
```

Create the values file:

```bash
export MENDER_SERVER_DOMAIN="mender.example.com"
umask 077
cat >mender-values.yml <<EOF
global:
  enterprise: true
  url: "https://${MENDER_SERVER_DOMAIN}"

default:
  imagePullSecrets:
    - name: my-mender-pull-secret

ingress:
  enabled: true
  ingressClassName: traefik
  path: /
  hosts:
    - ${MENDER_SERVER_DOMAIN}
  tls:
    - secretName: mender-ingress-tls
      hosts:
        - ${MENDER_SERVER_DOMAIN}
  selfSignedTLS:
    enabled: true

rustfs:
  enabled: true

mongodb:
  auth:
    rootPassword: "$(openssl rand -hex 24)"
  persistence:
    enabled: true

redis:
  enabled: true
EOF
```

Install the Mender Server:

```bash
helm repo add mender https://charts.mender.io
helm repo update
helm upgrade --install mender mender/mender --wait --timeout 15m -f mender-values.yml
```

Create the first organization and its administrator:

```bash
kubectl exec deploy/mender-tenantadm -- tenantadm create-org --name "replace-with-your-org" \
  --username "replace-with-your-email" --password "replace-with-your-password" --plan enterprise
```

[/ui-tab]
[/ui-tabs]

## Trust the server certificate

The Helm chart signs the server certificate with its own certificate authority (CA).
Save the CA certificate:

```bash
kubectl get secret mender-ingress-tls -o jsonpath='{.data.ca\.crt}' | base64 -d > ca.crt
```

Devices and browsers trust the server once they trust `ca.crt`:

* On each device, copy `ca.crt` to `/etc/mender/server.crt` and set
  [`ServerCertificate`](../../../03.Client-installation/07.Configuration/50.Configuration-options/docs.md#servercertificate)
  to that path.
* On your workstation, import `ca.crt` into the browser or system trust store.

The certificate is valid for 800 days and is kept on upgrades. To replace it,
delete the `mender-ingress-tls` Secret, run `helm upgrade` again, and distribute
the new `ca.crt`.

Open `https://mender.example.com`, replacing the name with your domain, and log in.

## Remove the installation

This removes K3s together with all the data:

```bash
/usr/local/bin/k3s-uninstall.sh
```
