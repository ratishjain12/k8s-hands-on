# Pod

Exercise 1. Manifest: [labs/01-core-objects/pod/pod.yaml](../../labs/01-core-objects/pod/pod.yaml)

A Pod is the smallest deployable unit: one or more containers sharing a network namespace (localhost, one IP) and optionally volumes.

## Apply and inspect

```bash
kubectl apply -f labs/01-core-objects/pod/pod.yaml
kubectl get pods -w
kubectl describe pod nginx-pod
```

Reading the Events section of `describe` shows which component did what:

| Event                 | Component                                                                                                 |
| --------------------- | --------------------------------------------------------------------------------------------------------- |
| `Scheduled`           | default-scheduler picked a node                                                                           |
| `Pulling` / `Pulled`  | kubelet asked containerd to pull the image (23s here, which is why the status sat at `ContainerCreating`) |
| `Created` / `Started` | kubelet started the container                                                                             |

There is no controller-manager step, because a bare Pod has no controller behind it.

Other things seen in `describe`:

- **Pod IP** (`10.244.0.5`) comes from the node's pod CIDR via the CNI (kindnet). It is different from the node IP.
- **Injected volume** `kube-api-access-...` holds a service account token so the Pod can call the API server. It was added automatically.
- **QoS class BestEffort**, because no requests or limits were set. BestEffort Pods are evicted first under node pressure. Requests are checked against Allocatable (see [cluster-components.md](cluster-components.md)).
- **Default tolerations** for `not-ready` and `unreachable` with a 300s window were added automatically.

## Reaching a Pod without a Service

```bash
kubectl port-forward pod/nginx-pod 8080:80
curl localhost:8080
```

`port-forward` tunnels through the API server to one specific Pod. It is a debugging tool: it stops with the terminal session, and it is bound to that Pod, not to any app. `containerPort` in the manifest is informational only and does not open a port.

## Deleting a Pod

`kubectl delete pod nginx-pod` removes the Pod object from etcd. It does not come back because nothing owns it. No controller holds a higher-level desired state like "N replicas of this should exist".

Check ownership:

```bash
kubectl get pod nginx-pod -o yaml | grep -A5 ownerReferences   # empty for a bare Pod
```

A Pod created by a Deployment is owned by a ReplicaSet, and that is what recreates it (Exercise 2).

## restartPolicy vs. deleting

- `restartPolicy: Always` (the default) is handled by the **kubelet**. It restarts a crashed _container_ inside the Pod and increments `RESTARTS`.
- It does not recreate a deleted _Pod_. That needs a controller.

Experiment to tell the two cases apart:

```bash
docker exec learn-control-plane crictl ps            # find the nginx container ID
docker exec learn-control-plane crictl stop <id>     # kill the container
kubectl get pods                                      # watch RESTARTS
kubectl delete pod nginx-pod                          # then delete the Pod
```

## Init and sidecar containers

Both live in the same Pod and share its network and volumes. They differ in when they run.

- **Init container** (`initContainers`): runs before the main containers, one at a time, each to completion. If one fails, the main containers do not start. Uses: wait for a dependency, run a migration, download files into a shared volume.
- **Sidecar**: runs alongside the main container for the Pod's whole life. Uses: log shipper, service-mesh proxy. Since Kubernetes 1.29 a native sidecar is an init container with `restartPolicy: Always`, which starts before the main container, keeps running, and shuts down after it.

```yaml
spec:
  initContainers:
    - name: wait-for-db
      image: busybox
      command: ["sh", "-c", "until nc -z db 5432; do sleep 1; done"]
    - name: log-shipper
      image: fluent-bit
      restartPolicy: Always
  containers:
    - name: app
      image: my-app
```

## Habits

- Pin image tags (`nginx:1.27`, not `latest`) so results are reproducible.
- Reach for `kubectl explain pod.spec.containers` before the docs.
