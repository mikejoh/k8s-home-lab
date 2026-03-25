# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Single Intel NUC k3s homelab cluster. This repo is the **infrastructure-as-code store** — actual applications are in a separate repo ([mikejoh/k8s-applications](https://github.com/mikejoh/k8s-applications)) and deployed via ArgoCD ApplicationSet.

## Architecture

- **k3s** with flannel, servicelb, and traefik disabled
- **Cilium** as CNI with kube-proxy replacement, L2 announcements, and Gateway API support
- **ArgoCD** manages application deployment via an ApplicationSet (`argo-cd/appset.yaml`) that pulls app definitions from the external `k8s-applications` repo using a matrix generator (git + list)
- **Tailscale** exposes services externally via Ingress resources with auto-TLS (manifests/)
- **Cilium Gateway** handles internal HTTP routing (`cilium/gateway.yaml`)
- **system-upgrade-controller** automates k3s version upgrades (`k3s/server-plan.yaml`)

## Repository Structure

- `argo-cd/` — ArgoCD ApplicationSet and HTTPRoute for the ArgoCD UI
- `cilium/` — Cilium Helm values and Gateway resource
- `k3s/` — k3s upgrade Plan for system-upgrade-controller
- `manifests/` — Tailscale Ingress resources (ArgoCD, Grafana, Prometheus)
- `tls/` — Client certificates for kubeconfig (gitignored)

## Common Commands

There are no build scripts or Makefiles. Deployment is manual:

```bash
# Install/upgrade Cilium
helm upgrade --install cilium cilium/cilium --namespace kube-system -f cilium/values.yaml

# Install/upgrade ArgoCD
helm upgrade --install argocd argo-cd/argo-cd --namespace argocd --create-namespace

# Apply manifests
kubectl apply -f argo-cd/appset.yaml
kubectl apply -f cilium/gateway.yaml
kubectl apply -f manifests/

# Temporary ArgoCD UI access
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

## Key Patterns

- **ApplicationSet matrix generator**: Combines a git generator (reads `apps/apps.yaml` from k8s-applications repo) with a list generator. Each app gets its own ArgoCD Application with configurable autosync.
- **No secrets management tooling**: Secrets are handled outside this repo.
- **Tailscale Ingress**: Uses `ingressClassName: tailscale` with HTTPS backends for service exposure.
- **Cilium values** target device `eno1` and cluster endpoint `192.168.1.207:6443`.
