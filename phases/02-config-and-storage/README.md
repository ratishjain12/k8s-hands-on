# Phase 02: Config and storage

## Goals

By the end you can:

- Separate configuration from container images using ConfigMaps and Secrets.
- Explain exactly when a running Pod sees a changed ConfigMap, and when it doesn't.
- Explain what a Secret does and does not protect.
- Request persistent storage with a PVC, and explain what happens to the data when the Pod, the PVC, and the PV are deleted.
- Run a stateful workload with a StatefulSet and explain how it differs from a Deployment.

## Concepts

- **ConfigMap:** non-sensitive configuration as key-value pairs. Consumed as environment variables or as files.
- **Secret:** the same shape, intended for sensitive values. Base64-encoded, not encrypted.
- **Volume:** a directory made available to the containers in a Pod. It lives as long as the Pod, or longer, depending on its type.
- **emptyDir:** scratch space that is created with the Pod and deleted with it. Shared by all containers in the Pod.
- **PersistentVolume (PV):** a piece of storage in the cluster.
- **PersistentVolumeClaim (PVC):** a request for storage by a Pod's owner. It binds to a PV.
- **StorageClass:** how PVs are created on demand. A provisioner does the work. The `volumeBindingMode` and `reclaimPolicy` decide when it happens and what happens at the end.
- **StatefulSet:** like a Deployment, but each Pod has a stable name, a stable DNS name through a headless Service, and its own PVC.

## Tools

- `kubectl create configmap | secret` (imperative, to see the shape of what you will write by hand)
- `kubectl get sc | pv | pvc`
- `kubectl exec` to look inside a running container
- `kubectl get secret -o yaml` and `base64 -d`
- `kubectl explain pod.spec.volumes`

## Reading

- Kubernetes docs: Concepts > Configuration (ConfigMaps, Secrets), Storage (Volumes, Persistent Volumes, Storage Classes), Workloads > StatefulSets
- `kubectl explain statefulset.spec.volumeClaimTemplates`

## Files

- [exercises.md](exercises.md): tasks and questions
- [hints.md](hints.md): open only when stuck
