# Phase 02 hints

Try for at least 10 minutes before opening a hint. Hints are ordered from nudge to near-answer.

## 0. Explore storage

1. `kubectl get sc` marks the default with `(default)` next to the name.
2. `kubectl describe sc standard` shows the provisioner, reclaim policy, and binding mode.

## 1. ConfigMap as environment variables

1. `kubectl explain pod.spec.containers.env.valueFrom.configMapKeyRef` is the single-key form. `kubectl explain pod.spec.containers.envFrom` is the bulk form.
2. Environment variables are fixed when a container starts. A running container's environment cannot change.
3. For a Pod, you recreate it. For a Deployment, look at `kubectl rollout restart`.

## 2. ConfigMap as files

1. You need two parts: a `volumes` entry of type `configMap` on the Pod, and a `volumeMounts` entry on the container. The ConfigMap keys become file names.
2. Mounted ConfigMap files are updated by the kubelet on a periodic sync, so expect a delay of up to about a minute, not an instant change.
3. A `subPath` mount is a single file bind-mounted from the volume. Ask what happens to a bind mount when the source is replaced.
4. A volume mounted at a path hides whatever the image had at that path. To add a single file without hiding the others, look at `subPath` again.

## 3. Secrets

1. `data` values must be base64-encoded by you. `stringData` takes plain text and the API server encodes it for you. Read both back with `get -o yaml` and compare, since `stringData` is write-only.
2. `base64 -d` decodes. On macOS, `base64 -D` also works with older versions.
3. As an environment variable the Secret appears in `env`. As a volume it appears as files, one per key.
4. Think about what is stored in etcd by default, and who can run `kubectl get secret`. RBAC (Phase 04) and encryption at rest are the real controls.

## 4. emptyDir

1. Two containers in one Pod can mount the same volume at different paths.
2. A volume's lifetime is tied to the Pod, not to a container.
3. A restarted container is in the same Pod, and a recreated Pod is a new one.

## 5. PersistentVolumeClaim

1. `kubectl explain pvc.spec` shows `accessModes`, `resources.requests.storage`, and `storageClassName`.
2. With `WaitForFirstConsumer`, the PV is not created until a Pod that uses the PVC is scheduled. The scheduler needs to know the node first.
3. `kubectl get pv` shows the PV that was created. `kubectl describe pv` shows where the path is. On kind, it is a directory on the node container, which you can look at with `docker exec learn-control-plane ls <path>`.
4. With reclaim policy `Delete`, deleting the PVC also deletes the PV and its data. `Retain` keeps them.
5. `kubectl describe pvc` and its Events show why a claim is stuck.

## 6. StatefulSet

1. A StatefulSet needs `serviceName` that refers to a headless Service (`clusterIP: None`). `kubectl explain statefulset.spec` lists the fields.
2. Pods are created in order, `<name>-0` first, and each waits for the previous one to be Ready (by default).
3. `volumeClaimTemplates` creates one PVC per Pod, named `<template-name>-<pod-name>`.
4. A Pod's DNS name is `<pod>.<service>.<namespace>.svc.cluster.local`.
5. Scaling down doesn't delete PVCs by default. Look at `persistentVolumeClaimRetentionPolicy` for the controls.
6. Deleting the StatefulSet doesn't delete its PVCs. Delete them yourself.
7. Think about what happens to a ReadWriteOnce volume when several Pods try to use it, and what "identity" means for a database replica.
