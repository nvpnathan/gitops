# Install Argo Workflow

## Pre-req's 

- Kubernetes Cluster
- Helmv3

## Install Artifactory

helm install argo-artifacts stable/minio \
  --set service.type=LoadBalancer \
  --set defaultBucket.enabled=true \
  --set defaultBucket.name=my-bucket \
  --set persistence.enabled=true \
  --set fullnameOverride=argo-artifacts

NOTE: When minio is installed via Helm, it uses the following hard-wired default credentials, which you will use to login to the UI:

AccessKey: AKIAIOSFODNN7EXAMPLE
SecretKey: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
Create a bucket named my-bucket from the Minio UI.

## Workflow Controller configmap

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

