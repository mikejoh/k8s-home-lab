# Kubernetes home lab

## Overview

* One Intel NUC
* k3s `v1.28.8+k3s1`
* Cilium as CNI

## Installation

`k3s` and the `config.yaml`:
```
cluster-init: true
write-kubeconfig-mode: "0644"
flannel-backend: "none"
disable-kube-proxy: true
disable-network-policy: true
disable:
  - servicelb
  - traefik
```
```
curl -sfL https://get.k3s.io | sh -s - --config=/etc/rancher/k3s/config.yaml
```

`cilium`:
```
helm upgrade \
  --install \
  --create-namespace \
  --namespace cilium \
  --debug \
  --version 1.15.3 \
  --set kubeProxyReplacement=true \
  --set k8sServiceHost=<nuc IP> \
  --set k8sServicePort=6443 \
  cilium \
  cilium/cilium
```

