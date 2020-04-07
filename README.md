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

Create CA secret and Cluster-issuer
```
kubectl apply -f ca-secret.yaml
kubectl apply -f ca-issuer.yaml
```

Verify that it is in a "READY" state
```
kubectl get clusterissuer                                                                            
NAME        READY   AGE
ca-issuer   True    70s
```

### Install Gitlab
```
kubectl create ns gitlab
kubens gitlab
kubectl apply -f gitlab-certificate.yaml
kubectl create secret generic gitlab-creds --from-literal=username=nness --from-literal=password='VMware1!'
helm install gitlab gitlab/gitlab -f gitlab-values.yaml
```

### Install Harbor
```
kubectl create ns harbor
kubens harbor
kubectl apply -f harbor-certificate.yaml
helm install harbor harbor/harbor \
  --set expose.type=loadBalancer \
  --set expose.tls.secretName=harbor-com-tls \
  --set expose.tls.commonName=harbor.vballin.com \
  --set externalURL=https://harbor.vballin.com \
  --set harborAdminPassword=VMware1!
```
### Install Argo
```
kubectl create ns argo
kubectl apply -f https://raw.githubusercontent.com/argoproj/argo/release-2.7/manifests/quick-start-postgres.yaml
kubectl edit svc argo-server
Change to type LoadBalancer
```

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

### Install Argo events

```
kubectl create ns argo-events
kubectl apply -f argo-events-install.yaml
kubectl apply -f https://raw.githubusercontent.com/argoproj/argo-events/master/hack/k8s/manifests/argo-events-cluster-roles.yaml
kubectl create secret generic gitlab-cert --from-file=vballin-ca.crt --from-file=gitlab-cert.crt
kubectl apply -f argo-webhook-eventsource.yaml
kubectl apply -f argo-webhook-gw.yaml
kubens argo
kubectl create sa argo-events-sa
kubectl apply -f argo-events-sa-binding.yaml
kubectl apply -f argo-webhook-go-reminders-sensor.yaml
```

### Install ArgoCD
```
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### (PKS ONLY) Workflow Controller configmap

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: workflow-controller-configmap
data:
  config: |
    dockerSockPath: /var/vcap/data/sys/run/docker/docker.sock
    artifactRepository:
      s3:
        bucket: my-bucket
        endpoint: argo-artifacts:9000
        insecure: true
        # accessKeySecret and secretKeySecret are secret selectors.
        # It references the k8s secret named 'argo-artifacts'
        # which was created during the minio helm install. The keys,
        # 'accesskey' and 'secretkey', inside that secret are where the
        # actual minio credentials are stored.
        accessKeySecret:
          name: argo-artifacts
          key: accesskey
        secretKeySecret:
          name: argo-artifacts
          key: secretkey
```

Add registry secret to default ns
```
kubens default
kubectl create secret docker-registry go-reminders-secrets --docker-server=harbor.vballin.com --docker-username=admin --docker-password=VMware1! --docker-email=nathan79@gmail.com
```
T/Sing
SSL is a nightmare
When submitting the accessToken secret for gitlab use --from-literal and not yaml
            while true; do sleep 30; done;
            echo '192.168.76.5 harbor.vballin.com' >> /etc/hosts &&
Ask why static serving files are not in the dockerfile
    - pointer to another repo


curl -ski -H "Content-Type: application/json" -H "PRIVATE-TOKEN: Bc8cPFYEY2PycLdLdcxL" -X GET https://gitlab.gl.vballin.com/api/v4/projects/4/hooks


curl -vv -ski -H "Content-Type: application/json" -H "PRIVATE-TOKEN: Bc8cPFYEY2PycLdLdcxL" -X POST https://gitlab.gl.vballin.com/api/v4/projects/4/hooks \
-d '{"url":"http://gitlab-gw.vballin.com:12000/push",
    "push_events":true,
    "enable_ssl_verification":false}'
