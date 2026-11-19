---
title: Demo Quick Start (Proposed)
taxonomy:
    category: docs
    label: tutorial
---

!! This page installs a single-node Mender Server for demo and evaluation
!! only. It uses a self-signed certificate, a single SeaweedFS instance and the
!! MongoDB bundled with the Helm chart. For production, follow the
!! [Kubernetes](../01.Kubernetes/docs.md), [Storage](../02.Storage/docs.md) and
!! [Mender Server](../03.Mender-server/docs.md) sections instead.

## Prerequisites

* A Linux host with at least 4 vCPUs and 8 GB RAM, with `curl`, OpenSSL 3 and root access.
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

## Create the TLS certificate

Create a local certificate authority (CA) and use it to sign the server certificate.
The files are readable only by you:

```bash
export MENDER_SERVER_DOMAIN="mender.example.com"
umask 077

openssl req -x509 -newkey ec -pkeyopt ec_paramgen_curve:P-256 -nodes -days 3650 \
  -subj "/CN=Mender demo CA" -keyout ca.key -out ca.crt

openssl req -x509 -newkey ec -pkeyopt ec_paramgen_curve:P-256 -nodes -days 365 \
  -subj "/CN=${MENDER_SERVER_DOMAIN}" \
  -addext "subjectAltName=DNS:${MENDER_SERVER_DOMAIN}" \
  -addext "basicConstraints=critical,CA:FALSE" \
  -addext "extendedKeyUsage=serverAuth" \
  -CA ca.crt -CAkey ca.key -keyout server.key -out server.crt

kubectl create secret tls mender-ingress-tls --cert=server.crt --key=server.key
```

Devices and browsers trust the server once they trust `ca.crt`:

* On each device, copy `ca.crt` to `/etc/mender/server.crt` and set
  [`ServerCertificate`](../../../03.Client-installation/07.Configuration/50.Configuration-options/docs.md#servercertificate)
  to that path.
* On your workstation, import `ca.crt` into the browser or system trust store.

Keep `ca.key` private. Anyone with it can create certificates your devices trust.

## Install the Artifact storage

Install SeaweedFS with an S3 endpoint and read the credentials it generates:

```bash
cat >seaweedfs.yml <<EOF
filer:
  s3:
    enabled: true
    enableAuth: true
s3:
  enabled: true
  enableAuth: true
EOF

helm repo add seaweedfs https://seaweedfs.github.io/seaweedfs/helm
helm repo update
helm install seaweedfs --wait --timeout 10m -f seaweedfs.yml seaweedfs/seaweedfs

export AWS_ACCESS_KEY_ID=$(kubectl get secret seaweedfs-s3-secret -o jsonpath='{.data.admin_access_key_id}' | base64 -d)
export AWS_SECRET_ACCESS_KEY=$(kubectl get secret seaweedfs-s3-secret -o jsonpath='{.data.admin_secret_access_key}' | base64 -d)
```

The Mender Server creates the `mender-artifact-storage` bucket on its first start.

## Install the Mender Server

The values file is readable only by you, because it contains credentials.
Keep it for later upgrades: MongoDB stores its data on a persistent volume and
keeps the root password it was first started with.

[ui-tabs position="top-left" active="0" theme="default" ]
[ui-tab title="Open Source"]

Create the values file:

```bash
umask 077
cat >mender-values.yml <<EOF
global:
  s3:
    AWS_URI: "http://seaweedfs-s3:8333"
    AWS_ACCESS_KEY_ID: "${AWS_ACCESS_KEY_ID}"
    AWS_SECRET_ACCESS_KEY: "${AWS_SECRET_ACCESS_KEY}"
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
umask 077
cat >mender-values.yml <<EOF
global:
  enterprise: true
  s3:
    AWS_URI: "http://seaweedfs-s3:8333"
    AWS_ACCESS_KEY_ID: "${AWS_ACCESS_KEY_ID}"
    AWS_SECRET_ACCESS_KEY: "${AWS_SECRET_ACCESS_KEY}"
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

Open `https://mender.example.com`, replacing the name with your domain, and log in.

## Remove the installation

This removes K3s together with all the data:

```bash
/usr/local/bin/k3s-uninstall.sh
```
