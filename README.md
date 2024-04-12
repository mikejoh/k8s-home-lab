# Kubernetes home lab

## Overview

* One Intel NUC
* k3s `v1.28.8+k3s1`
* Cilium as CNI

## Packages and tools on the NUC

* `fzf`:
```
git clone --depth 1 https://github.com/junegunn/fzf.git ~/.fzf
~/.fzf/install
```

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
  --set kubeProxyReplacement=strict \
  --set k8sServiceHost=<nuc IP> \
  --set k8sServicePort=6443 \
  --set gatewayAPI.enabled=true \
  --set operator.replicas=1 \
  --version 1.15.3 \
  cilium \
  cilium/cilium
```

### Create credentials to interact with the cluster with `kubectl`
_This assumes that you have a `tls` directory locally._
1. Run locally:
```
openssl genrsa -out nuc-admin.key 2048
openssl req -new -key nuc-admin.key -out nuc-admin.csr -subj /O=nuc-admin/CN=nuc-admin
cat nuc-admin.csr | base64 -w0 | wl-copy -p
```
2. Run externally (e.g. on the NUC), create the following manifest, i gave it the name `nuc-admin.yaml`. _Note that you can change `expirationSeconds` for longer validity, if you remove that completely you'll get the default 1 year validity from [the built-in signer](https://kubernetes.io/docs/reference/access-authn-authz/certificate-signing-requests/#kubernetes-signers)_:
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
3. Run externally:
```
kubectl apply -f nuc-admin.yaml
kubectl certificate approve nuc-admin
kubectl get csr nuc-admin -o jsonpath='{.status.certificate}' | base64 -d > admin.crt
```
4. Locally: Create a file called `nuc-admin.crt` locally in the `tls` directory.
5. Locally, finalize the `kubeconfig`:
```
kubectl config set-credentials nuc-admin --client-key nuc-admin.key --client-certificate nuc-admin.crt --embed-certs=true
kubectl config set-cluster <CLUSTER NAME> --server https://<NUC IP>:6443 --insecure-skip-tls-verify=true
kubectl config set-context nuc-admin --user=nuc-admin --cluster=<CLUSTER NAME>
kubectl config use-context nuc-admin
```

### Install ArgoCD

1. Install ArgoCD using Helm:
```
mkdir argo-cd
helm repo add argo https://argoproj.github.io/argo-helm
helm show values --version 6.7.3 argo/argo-cd > argo-cd/6.7.3-values.yaml
```
2. Make relevant changes to the values file.
3. Install:
```
helm upgrade \
  --install \
  --create-namespace \
  --namespace argocd \
  --values argo-cd/6.7.3-values.yaml \
  --version 6.7.3 \
  --debug \
  argocd \
  argo/argo-cd
```
4. Get the password set for the built-in `admin` account:
```
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 --decode ; echo
```
5. At the moment i'm only port-forwarding to my cluster services, so to be able to initially browse to the ArgoCD UI i did the following:
```
kubectl port-forward svc/argocd-server -n argocd 4443:443
```
I'll change the way i expose services and applications in the cluster later on.

5. Install the `ApplicationSet` to install all applications:
```
kubectl apply -f argo-cd/appset.yaml
```
