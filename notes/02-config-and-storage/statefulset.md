# StatefulSet

Exercise 6. Manifests: [labs/02-config-and-storage/statefulset/](../../labs/02-config-and-storage/statefulset/)

A StatefulSet is like a Deployment, but each Pod keeps its own identity and its own storage.

| | Deployment | StatefulSet |
|---|---|---|
| Pod names | Random suffix (`web-6f47-x2k9p`) | Ordinal (`web-0`, `web-1`, `web-2`) |
| A Pod is deleted | New Pod, new name | New Pod, **same name** |
| Storage | Shared, or none | Each Pod gets its **own PVC**, reattached to the same Pod |
| Start order | All at once | One at a time, in order |
| Network identity | Through a Service only | Each Pod has its own DNS name (headless Service) |

It exists for software that needs to know which instance it is: databases, Kafka, Zookeeper, Elasticsearch. It only provides the stable name and disk. Replication and leader election are still the software's job.

## The three pieces

```yaml
apiVersion: v1
kind: Service
metadata:
  name: headless-service
spec:
  clusterIP: None            # headless: no virtual IP
  selector:
    app: statefulset-app     # must match the Pod template labels
  ports:
    - name: http
      port: 80
```

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: statefulset-app
spec:
  serviceName: headless-service      # must be the headless Service's name
  replicas: 3
  selector:
    matchLabels:
      app: statefulset-app
  template:
    metadata:
      labels:
        app: statefulset-app
    spec:
      containers:
        - name: statefulset-container
          image: nginx:1.14.2
          volumeMounts:
            - name: statefulset-volume
              mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
    - metadata:
        name: statefulset-volume
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 100Mi
```

- A headless Service with no `selector` has no endpoints, so no DNS records. `ports` is optional for a headless Service, but is good practice.
- `serviceName` and the Service's `selector` are the two links most likely to be wrong. Check `kubectl get endpoints headless-service`. With the right selector it lists every Pod's IP (`10.244.0.61:80,10.244.0.63:80,10.244.0.64:80` here), and `<none>` means the selector doesn't match.
- `volumeClaimTemplates` is a PVC template. The StatefulSet creates one PVC per Pod.

## Observed names

Pods `statefulset-app-0`, `-1`, `-2`. PVCs follow `<template name>-<pod name>`:

```
statefulset-volume-statefulset-app-0
statefulset-volume-statefulset-app-1
statefulset-volume-statefulset-app-2
```

## Same name, same disk

Deleting `statefulset-app-0` brought back a Pod with the same name. Its Pod was 4m59s old while the PVC `...-0` was still 8m14s old. The PVC was not recreated, and the new Pod attached to the same one.

## Experiments on a throwaway StatefulSet

The rest was run on a separate StatefulSet (`tmp-sts`, busybox, volume `data` mounted at `/data`) so the lab's Pods and data were untouched. It was deleted afterwards.

**Start order.** Applied with 3 replicas. The Pods' ages were 20s, 14s, and 6s: `-0` started first, then `-1`, then `-2`.

**Data survives a Pod delete.** Each Pod wrote its hostname with `hostname > /data/who`. After deleting `tmp-sts-1`, the new Pod had the same name, a **different IP** (`10.244.0.68` before, `10.244.0.71` after), and `/data/who` still said `tmp-sts-1`.

**Scale 3 → 1 → 3.** The PVCs of the removed Pods were kept. When `tmp-sts-2` came back, its PVC was the original one (age 2m52s) and `/data/who` still held `tmp-sts-2`. Scaling down removes Pods but not their data.

**Deleting the StatefulSet does not delete the PVCs.** After `kubectl delete sts tmp-sts`, all three PVCs and PVs were still `Bound`. They have to be removed by hand with `kubectl delete pvc`. After that the PVs went `Released` and then disappeared (reclaim policy `Delete`, see [pvc.md](pvc.md)). Kubernetes has a `persistentVolumeClaimRetentionPolicy` field to change this behavior. That field is from the docs and wasn't tried here.

| Action | Pod | PVC and data |
|---|---|---|
| Delete a Pod | Returns, same name | Kept, reattached |
| Scale down | Removed | Kept |
| Scale back up | Returns, same name | Reattached |
| Delete the StatefulSet | Removed | **Kept** |

## DNS

```
statefulset-app-0.headless-service.default.svc.cluster.local
```

`<pod>.<service>.<namespace>.svc.<cluster domain>`. A normal Service name resolves to one virtual IP. A headless Service has no virtual IP, so DNS returns the Pods' own IPs. The `<pod>.<service>` form reaches **one specific Pod**, and the name keeps working when the Pod's IP changes.

Observed from another Pod, in the same namespace:

- `nslookup headless-service.default.svc.cluster.local` returned **all three** Pod IPs.
- `nslookup statefulset-app-0.headless-service.default.svc.cluster.local` returned one IP, `10.244.0.64`. That matches `kubectl get pods -o wide`.
- `ping statefulset-app-0.headless-service` worked and reached `10.244.0.64`.
- `ping headless-service` worked and reached one of the Pod IPs (`10.244.0.64`).

**`nslookup` gotcha.** `nslookup statefulset-app-0.headless-service` returned `NXDOMAIN` on busybox, but `ping` with the same name worked. The Pod's `resolv.conf` has `search default.svc.cluster.local svc.cluster.local cluster.local` and `ndots:5`, so the short name should expand. Busybox's `nslookup` most likely doesn't apply the search list (this was not confirmed in its source). When a short name fails in `nslookup`, retry with the full name or with another tool (`ping`, `wget`) before suspecting DNS.

## Commands

```bash
kubectl get pods,pvc -w
kubectl get endpoints headless-service
kubectl scale sts/<name> --replicas=1
kubectl rollout status sts/<name>
kubectl delete sts <name>
kubectl delete pvc <name>
kubectl exec <pod> -- nslookup <name>.<namespace>.svc.cluster.local
```
