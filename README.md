# Kubernetes home lab

## Overview

* One Intel NUC
* k3s `v1.31.9+k3s1`
* Cilium as CNI
* ArgoCD for GitOps
* Tailscale operator for service exposure and cluster access

## Packages and tools on the NUC

* `fzf`:

```bash
git clone --depth 1 https://github.com/junegunn/fzf.git ~/.fzf
~/.fzf/install
```

## Installing `k3s`

`k3s` config at `/etc/rancher/k3s/config.yaml`:

```yaml
cluster-init: true
write-kubeconfig-mode: "0644"
flannel-backend: "none"
disable-kube-proxy: true
disable-network-policy: true
disable:
  - servicelb
  - traefik
```

```bash
curl -sfL https://get.k3s.io | sh -s - --config=/etc/rancher/k3s/config.yaml
```

## Install Cilium

```bash
helm upgrade \
  --install \
  --create-namespace \
  --namespace kube-system \
  --reuse-values \
  -f cilium/values.yaml \
  --version 1.16.4 \
  cilium \
  cilium/cilium
```

## Install ArgoCD

```bash
helm upgrade \
  --install \
  --reuse-values \
  --create-namespace \
  --namespace argocd \
  --values argo-cd/values.yaml \
  --version 7.7.7 \
  argocd \
  argo-cd/argo-cd
```

Get the initial `admin` password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 --decode; echo
```

Apply the ApplicationSet to deploy all apps:

```bash
kubectl apply -f argo-cd/appset.yaml
```

## Install the Tailscale operator

Follow the [Tailscale Kubernetes operator docs](https://tailscale.com/kb/1236/kubernetes-operator) to set up OAuth credentials and install the operator.

```bash
helm upgrade \
  --install \
  --create-namespace \
  --namespace tailscale \
  tailscale-operator \
  tailscale/tailscale-operator
```

## Cluster access via `kubectl`

Cluster access uses the **Tailscale API server proxy** — no client certificates, no expiry.

### Prerequisites

* Tailscale installed and authenticated on your local machine
* Your device has the `tag:k8s-admin` tag assigned in the Tailscale admin console

To assign the tag, add this to your Tailscale ACL policy:

```json
"tagOwners": {
  "tag:k8s-admin": ["autogroup:admin"]
}
```

Then go to **Machines** → your machine → **Edit tags** → add `tag:k8s-admin`.

The `tag:k8s-admin` group is bound to `cluster-admin` via a `ClusterRoleBinding` managed by ArgoCD.

### Generate kubeconfig

```bash
tailscale configure kubeconfig tailscale-operator
kubectl --context=tailscale-operator get nodes
```

The proxy runs on the `tailscale-operator` Tailscale device. Access is gated by Tailscale device identity — only devices tagged `tag:k8s-admin` are granted cluster-admin.

### Emergency access

If the Tailscale proxy is unavailable, SSH to the NUC and use the local k3s kubeconfig:

```bash
ssh <user>@nuc01
sudo kubectl --kubeconfig=/etc/rancher/k3s/k3s.yaml get nodes
```

## Expose services over Tailscale

Services are exposed using `Ingress` resources with `ingressClassName: tailscale`, providing auto-TLS and stable `<name>.<tailnet>.ts.net` URLs. Ingress manifests are in `manifests/`.

Example:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: prometheus
  namespace: kube-prometheus-stack
spec:
  ingressClassName: tailscale
  rules:
    - host: prometheus
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: kube-prometheus-stack-prometheus
                port:
                  number: 9090
  tls:
    - hosts:
        - prometheus
```

## Upgrade k3s

Uses `system-upgrade-controller`. Install it:

```bash
export SUC_VERSION="v0.14.2"
kubectl apply --force-conflicts --server-side -f https://github.com/rancher/system-upgrade-controller/releases/download/${SUC_VERSION}/crd.yaml
kubectl apply -f https://github.com/rancher/system-upgrade-controller/releases/download/${SUC_VERSION}/system-upgrade-controller.yaml
```

To upgrade k3s, update the version in `k3s/server-plan.yaml` and apply:

```bash
kubectl apply -f ./k3s/server-plan.yaml
```
