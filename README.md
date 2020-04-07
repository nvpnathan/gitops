# Install Argo Workflow

## Pre-req's 

- Kubernetes Cluster
    - Privileged containers
    - Persistant Storage
    - Default StorageClass
    - Service Type: LoadBalancer
- Helmv3 CLI
- Argo CLI
- ArgoCD CLI
- SSH key
- kubectx/kubens

## Create privileged psp policy
tkc-rbac.yaml
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: psp:privileged
rules:
- apiGroups: ['policy']
  resources: ['podsecuritypolicies']
  verbs:     ['use']
  resourceNames:
  - vmware-system-privileged
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: all:psp:privileged
roleRef:
  kind: ClusterRole
  name: psp:privileged
  apiGroup: rbac.authorization.k8s.io
subjects:
- kind: Group
  name: system:serviceaccounts
  apiGroup: rbac.authorization.k8s.io
```
```
kubectl apply -f tkc-rbac.yaml
```

## Install Cert-Manager
```
kubectl create ns cert-manager
```

Create namespace for cert-manager
```
kubens cert-manager
```
Install cert-manager
```
kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v0.14.1/cert-manager.yaml
```

Create a Root CA cert.

```
kubectl apply -f ca-secret.yaml
kubectl apply -f ca-issuer.yaml
```

## Install Artifact repo

```
helm install argo-artifacts stable/minio \
  --set service.type=LoadBalancer \
  --set defaultBucket.enabled=true \
  --set defaultBucket.name=my-bucket \
  --set persistence.enabled=false \
  --set fullnameOverride=argo-artifacts
```

### Install Gitlab
helm install gitlab gitlab/gitlab \
  --set global.hosts.domain=gl.vballin.com \
  --set global.edition=ce \
  --set global.initialRootPassword.secret=gitlab-creds \
  --set global.initialRootPassword.key=password \
  --set certmanager-issuer.email=nathan79@gmail.com

### Install Harbor
helm install harbor harbor/harbor \
  --set expose.type=loadBalancer \
  --set expose.tls.commonName=harbor-k8s.vballin.com \
  --set externalURL=https://harbor-k8s.vballin.com \
  --set harborAdminPassword=VMware1!


NOTE: When minio is installed via Helm, it uses the following hard-wired default credentials, which you will use to login to the UI:

AccessKey: AKIAIOSFODNN7EXAMPLE
SecretKey: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY

```
kubectl create rolebinding default-admin --clusterrole=admin --serviceaccount=argo:default
```

```
kubectl create secret generic gitlab-creds --from-literal=username=nness --from-literal=password='VMware1!'
kubectl create secret generic gitlab-cert --from-file=vballin-ca.crt --from-file=gitlab-cert.crt
kubectl create secret generic gitlab-ssh-creds --from-file=/Users/nness/.ssh/id_rsa
kubectl create secret generic harbor-creds --from-literal=username=admin --from-literal=password='VMware1!'
kubectl create secret generic harbor-cert --from-file=ca.crt
```

T/Sing
SSL is a nightmare
When submitting the accessToken secret for gitlab use --from-literal and not yaml

Workflow T/Sing

while true; do sleep 30; done;

echo '192.168.76.5 harbor.vballin.com' >> /etc/hosts &&

Ask why static serving files are not in the dockerfile
    - pointer to another repo


curl -vv -ski -H "Content-Type: application/json" -H "PRIVATE-TOKEN: Bc8cPFYEY2PycLdLdcxL" -X POST https://gitlab.gl.vballin.com/api/v4/projects/4/hooks \
-d '{"url":"http://gitlab-gw.vballin.com:12000/push",
    "push_events":true,
    "enable_ssl_verification":false}'
