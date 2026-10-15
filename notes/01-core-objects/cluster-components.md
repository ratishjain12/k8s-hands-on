# Cluster components

Explored with `kubectl get pods -A`, `kubectl describe node`, `docker ps`, and `docker exec learn-control-plane crictl ps` on a kind cluster.

## Control plane

| Component | Role |
|---|---|
| kube-apiserver | Front door of the cluster. Handles authentication, validation, and the REST API. The only component that reads and writes etcd. It makes no scheduling or reconciliation decisions. All other components talk to each other through it. |
| etcd | The cluster's database, a key-value store holding all desired and current state. Every `kubectl apply` ends up as a record here. |
| kube-scheduler | Picks a node for Pods that have none assigned. Does not start containers. |
| kube-controller-manager | Runs the reconciliation loops (Deployment, ReplicaSet, Node controllers, and others). Recreates a Pod that was deleted, and creates ReplicaSets for Deployments. |

## On every node

| Component | Role |
|---|---|
| kubelet | A process on the node, not a Pod. Watches the API server for Pods assigned to its node and starts them through the container runtime. Reports status back. |
| kube-proxy | Programs iptables/IPVS rules so traffic sent to a Service IP reaches one of the Service's Pods. This is what makes Services work. |
| containerd | The container runtime. Kubernetes talks to it over CRI. |

## Add-ons

| Component | Role |
|---|---|
| CoreDNS | Cluster DNS. Resolves names like `my-svc.default.svc.cluster.local` to Service IPs. |
| kindnet | kind's CNI plugin. Gives Pods IPs and connects them across nodes. |
| local-path-provisioner | kind's default storage provisioner (namespace `local-path-storage`). Creates volumes for PersistentVolumeClaims. Relevant in phase 02. |

## What happens on `kubectl apply` of a Deployment

Written from the component roles above, to be checked against Exercise 2.

1. kubectl sends the manifest to the **API server**, which validates it and stores it in **etcd**.
2. The **controller-manager** sees the new Deployment and creates a ReplicaSet, which creates Pod objects with no node assigned.
3. The **scheduler** sees unassigned Pods and assigns each to a node.
4. The **kubelet** on that node sees a Pod assigned to it and starts the containers through **containerd**.

## kind specifics

kind runs each node as a Docker container. Pods run inside that container under containerd, so they do not appear in `docker ps`.

```
your Mac
 └─ Docker                      ← docker ps sees this level
     └─ learn-control-plane     ← the kind "node" (a container)
         └─ containerd          ← crictl ps sees this level
             └─ etcd, coredns, nginx ...
```

Useful commands:

```bash
docker exec learn-control-plane crictl ps      # containers inside the node
docker exec learn-control-plane crictl ps -a   # includes exited containers
```

On a real node, run `crictl` directly on the machine. This is the fallback when `kubectl` can't explain why a container won't start.

## Capacity vs. Allocatable

Capacity is the node's total resources. Allocatable is what the scheduler may hand out to Pods:

```
Allocatable = Capacity − kube-reserved − system-reserved − eviction threshold
```

The reservations keep room for the kubelet, container runtime, OS, and an eviction buffer, so Pods can't starve the node itself. The scheduler checks Pod requests against Allocatable only. On kind the two are close to identical because no reservations are configured. On EKS the gap is noticeable on small instances.

## If etcd dies

**Keeps working:** Pods already running. The kubelet and containerd run containers without needing etcd, and kube-proxy rules already on the nodes keep routing Service traffic. The data plane is independent of the control plane.

**Stops working:** anything that needs cluster state.

- The API server can't persist or reliably read state, so `kubectl` calls fail or error out.
- No new Pods, deployments, scaling, or config changes.
- No self-healing. Controllers can't act, so a Pod that dies is not replaced, and node failures aren't recorded.
- A node that fails during the outage leaves its Pods unrecoverable until etcd is back.

**Recovery:** restore etcd from a snapshot. If the data is lost with no backup, the cluster no longer knows what it was supposed to run, even though containers may still be running. This is why etcd is run as a 3 or 5 member quorum in production (it tolerates a minority of members failing), and why snapshots are taken regularly. On EKS, AWS runs and backs up etcd for you.
