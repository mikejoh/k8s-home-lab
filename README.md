# Kubernetes home lab

## Overview

* One Intel NUC
* k3s `v1.28.8+k3s1`
* Cilium as CNI

## Installing `k3s`

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

### Install `cilium`

`cilium`:
```
helm upgrade \
  --install \
  --create-namespace \
  --namespace cilium \
  --debug \
  --set kubeProxyReplacement=true \
--set k8sServiceHost=<nuc IP> \
  --set k8sServicePort=6443 \
  --set operator.replicas=1 \
  --version 1.15.3 \
  cilium \
  cilium/cilium
```

### To manage the `k3s` cluster with `kubectl` externally you can do the following:
_This assumes that you have a `tls` directory locally._
1. Locally: `openssl genrsa -out tls/nuc-admin.key 2048`
2. Locally: `openssl req -new -key nuc-admin.key -out tls/nuc-admin.csr -subj /O=nuc-admin/CN=nuc-admin`
3. Locally: Copy-paste the following and add that to the manifest below: `cat nuc-admin.csr | base64 -w0`
3. Externally, create the following manifest on the NUC, i gave it the name `nuc-admin.yaml`:
```
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: nuc-admin
spec:
  groups:
  - nuc-admin
  request: <COPY-PASTE THE B64 ENCODED CSR HERE>
  signerName: kubernetes.io/kube-apiserver-client
  expirationSeconds: 108000
  usages:
  - client auth
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: nuc-admin
```
4. Externally: `kubectl apply -f nuc-admin.yaml`
5. Externally: `kubectl certificate approve nuc-admin`
6. Externally: Copy-paste the certificate from this command: `kubectl get csr nuc-admin -o jsonpath='{.status.certificate}' | base64 -d > admin.crt`
7. Locally: Create a file called `nuc-admin.crt` locally in the `tls` directory.
8. Locally, finalize the `kubeconfig`:
```
kubectl config set-credentials nuc-admin --client-key tls/nuc-admin.key --client-certificate tls/nuc-admin.crt --embed-certs=true
kubectl config set-cluster <CLUSTER NAME> --server https://<NUC IP>:6443 --insecure-skip-tls-verify=true
kubectl config set-context nuc-admin --user=nuc-admin --cluster=<CLUSTER NAME>
kubectl config use-context nuc-admin
```

