# Phase 02 solutions: debug challenge

**Spoilers.** Open this only after you have diagnosed each case in `labs/02-config-and-storage/broken/`, or after you have been stuck for a long time.

## 01-configmap-env

**Symptom:** the Pod is stuck in `CreateContainerConfigError`.

**How to find it:** `kubectl describe pod` shows `Error: configmap "app-settings" not found`. `kubectl get configmap -n debug-02` lists `app-setting`.

**Root cause:** the Deployment refers to a ConfigMap called `app-settings`. The ConfigMap in the file is called `app-setting`. The name in `configMapKeyRef.name` doesn't match.

**Fix:** use the same name in both places.

## 02-configmap-file

**Symptom:** the Pod ends up in `RunContainerError` and restarts.

**How to find it:** `kubectl describe pod` shows a mount error ending in `not a directory`. Compare `mountPath` with `subPath` in the manifest.

**Root cause:** `subPath: index.html` mounts a single file, but `mountPath` is the directory `/usr/share/nginx/html`. A file can't be mounted onto a directory.

**Fix:** `mountPath: /usr/share/nginx/html/index.html`. Alternatively, drop `subPath` and mount the whole ConfigMap, which hides the other files in the directory.

## 03-secret-volume

**Symptom:** `kubectl apply` is rejected with `strict decoding error: unknown field "spec.template.spec.volumes[0].secret.name"`. The Secret in the same file was still created.

**Root cause:** a `secret` volume names its Secret with `secretName`. `name` is the field a `configMap` volume uses.

**Fix:** `secretName: tls-03`.

## 04-emptydir

**Symptom:** `kubectl apply` is rejected with `spec.containers[1].volumeMounts[0].name: Not found: "shared"`.

**Root cause:** the `reader` container mounts a volume called `shared`, but the Pod's only volume is called `scratch`. Every `volumeMounts[].name` has to match a name under `volumes`.

**Fix:** use `scratch` in both containers.

## 05-pvc

**Symptom:** the Pod and the PVC both stay `Pending`.

**How to find it:** `kubectl describe pvc data-05 -n debug-02` shows a `ProvisioningFailed` event: `NodePath only supports ReadWriteOnce and ReadWriteOncePod (1.22+) access modes`. The Pod's own events say nothing useful, so start from the claim.

**Root cause:** the claim asks for `ReadWriteMany`, and the `standard` class (local-path) can only provide `ReadWriteOnce`.

**Fix:** `ReadWriteOnce`. A PVC's spec is immutable after creation, so applying the fixed file is rejected. Delete the Pod and the PVC, then apply again.

## 06-statefulset

**Symptom:** both Pods are `Running`, but `kubectl get endpoints store-06 -n debug-02` shows `<none>`, and neither `store-06` nor `store-0.store-06` resolves.

**Root cause:** the headless Service selects `app: stores`, but the StatefulSet's Pods are labeled `app: store`. With no matching Pods the Service has no endpoints, so DNS has no records to return.

**Fix:** `app: store` in the Service selector. This is the same bug as the Service selector typo in Phase 01. The headless Service is more dangerous here because the Pods look healthy.
