# Deployment

Exercises 2 and 3. Manifest: [labs/01-core-objects/deployment/deployment.yaml](../../labs/01-core-objects/deployment/deployment.yaml)

A Deployment declares "N replicas of this Pod template should exist" and manages ReplicaSets to get there. It adds rolling updates and rollback on top of a ReplicaSet, and a ReplicaSet adds self-healing on top of a bare Pod.

```
Deployment → ReplicaSet → Pods
```

## Labels, selectors, containerPort

```yaml
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx        # which Pods this Deployment owns
  template:
    metadata:
      labels:
        app: nginx      # label stamped on every Pod it creates
```

- **Labels** are key-value tags on any object. They mean nothing by themselves.
- **Selector** is the query that finds objects by label. The Deployment uses it to count its Pods. A Service uses the same mechanism to find backends.
- `selector` must match the template labels, or the API rejects the Deployment. It is immutable after creation.
- **containerPort** is informational only. It does not open or restrict a port. nginx listens on 80 whatever is declared. What decides routing is a Service's `targetPort`. Declaring a `name` lets a Service refer to the port by name.

Useful commands:

```bash
kubectl get pods --show-labels
kubectl get pods -l app=nginx
kubectl describe pods -l app=nginx          # describe every Pod matching the selector
```

## Scaling

```bash
kubectl scale deployment/nginx-deployment --replicas=5
```

`scale` is imperative. It changes the live Deployment but not the file.

Observed: after `scale` to 5 there were 5 Pods. After `kubectl apply` of the unchanged file there were 3 again, because the file says `replicas: 3` and `apply` makes the live object match it.

`apply` compares the file, the live object, and the `last-applied-configuration` annotation, and only touches fields that appear in the file. If the manifest had no `replicas` field, a `scale` would survive the next `apply`. This is how HPA is used alongside `apply` (Phase 4): leave `replicas` out of the manifest.

## ReplicaSets and Pod names

Pod names look like `nginx-deployment-6f4765847d-cvp66`: Deployment name, ReplicaSet hash, random suffix. The hash is of the Pod template (the `pod-template-hash` label), so changing the template creates a new ReplicaSet.

```bash
kubectl get rs
```

## Rolling update with a bad image

Changed `image` to a tag that does not exist (`nginx:1.99.99`) and applied.

Observed (`kubectl get pods`):

```
nginx-deployment-5497c6cd97-pjmxn   0/1   ImagePullBackOff   ...   # new ReplicaSet
nginx-deployment-6f4765847d-cvp66   1/1   Running            ...   # old ReplicaSet, untouched
nginx-deployment-6f4765847d-fx448   1/1   Running
nginx-deployment-6f4765847d-h8bzj   1/1   Running
```

- The new Pod alternated between `ErrImagePull` (a failed attempt) and `ImagePullBackOff` (waiting before the retry). The wait grows up to a 5-minute cap.
- The 3 old Pods kept running, so there was no downtime.
- Why: the default strategy is `maxSurge: 25%` and `maxUnavailable: 25%`. With 3 replicas, surge rounds up to 1 and unavailable rounds down to 0. The Deployment may add one Pod, but may not remove an old one until the new Pod is `Ready`. The new Pod never became Ready, so the rollout stalled safely.

Commands to inspect it:

```bash
kubectl describe deployment nginx-deployment     # RollingUpdateStrategy, replica counts
kubectl rollout status deployment/nginx-deployment
kubectl get rs
kubectl describe pod <failing-pod>               # Events show the reason
kubectl get events --sort-by=.lastTimestamp
```

## Self-healing

Deleting a Pod owned by a Deployment: the ReplicaSet sees fewer Pods than `replicas` and creates a replacement. Compare with a bare Pod, which nothing recreates (see [pod.md](pod.md)).

```bash
kubectl get pod <name> -o yaml | grep -A5 ownerReferences   # shows the owning ReplicaSet
```
