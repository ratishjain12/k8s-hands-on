# Service

Exercise 4. Manifest: [labs/01-core-objects/services/service.yaml](../../labs/01-core-objects/services/service.yaml)

Pods are ephemeral and get a new IP each time they are recreated, so clients can't depend on Pod IPs. A Service gives a changing set of Pods a **stable virtual IP and DNS name**, and load-balances across them.

## Manifest

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
```

- `selector` picks the backend Pods by label. It must match the labels on the **Pod template** (`spec.template.metadata.labels` in the Deployment), not the Deployment's own labels.
- `port` is the port the Service listens on.
- `targetPort` is the port on the Pod that traffic is sent to. It defaults to `port`, which is why omitting it works here: nginx listens on 80. If the container listened on 8080, this would be `port: 80` with `targetPort: 8080`. It can also be a named container port (`targetPort: http`), so the Service doesn't need to know the number.
- `type` defaults to `ClusterIP`.

## How it works

```
client Pod ──► nginx-service (ClusterIP, virtual)
                    │  kube-proxy rules on the node rewrite the destination
                    ▼
              one of the Pod IPs (10.244.0.7 / .8 / .9)
```

1. **Endpoints.** The control plane watches the Pods matching the selector and records their IPs as the Service's endpoints. Only Pods that are **Ready** are included, so a Pod that fails its readiness check stops receiving traffic without being killed.
2. **kube-proxy** runs on every node and programs iptables/IPVS rules so that traffic to the Service IP is sent to one of the endpoints. The ClusterIP isn't a real interface or process. It exists only as these rules. Load balancing is per connection and roughly random, not round-robin.
3. **CoreDNS** resolves the Service name to the ClusterIP.

The ClusterIP comes from a separate range from the Pod CIDR (Pods here are `10.244.x.x`).

Observed:

```
$ kubectl get endpoints
NAME            ENDPOINTS                                   AGE
kubernetes      192.168.155.2:6443                          136m
nginx-service   10.244.0.7:80,10.244.0.8:80,10.244.0.9:80   12s
```

The three endpoints were the three Deployment Pod IPs. The `kubernetes` Service is created by the cluster itself. Its endpoint is the API server, and it is how Pods reach the API.

`Endpoints` is deprecated since v1.33 in favour of **EndpointSlice**, which splits large endpoint lists into chunks so a Service with thousands of Pods doesn't produce one huge object.

```bash
kubectl get endpointslices
```

## Service DNS

```
<service>.<namespace>.svc.cluster.local
```

- Within the same namespace, the short name `nginx-service` works, because Pod DNS config includes the namespace as a search domain.
- From another namespace, use at least `<service>.<namespace>`.
- Services that exist when a Pod starts also get injected as environment variables (e.g. `NGINX_SERVICE_SERVICE_HOST`). DNS is the reliable way, since it works for Services created later.

### How a name resolves

Kubernetes writes `/etc/resolv.conf` into every Pod. It looks like:

```
nameserver <ClusterIP of the kube-dns Service, which fronts CoreDNS>
search <pod-namespace>.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

With `ndots:5`, a name with fewer than 5 dots is tried with each `search` suffix first. From a Pod in `default`:

| Name used | Tried as | Reaches |
|---|---|---|
| `nginx-service` | `nginx-service.default.svc.cluster.local` | the `default` Service |
| `nginx-service.staging` | `nginx-service.staging.default.svc.cluster.local` (no such name), then `nginx-service.staging.svc.cluster.local` | the `staging` Service |
| `nginx-service.staging.svc.cluster.local` | as written | the `staging` Service |

CoreDNS watches the API server for Services and returns the Service's ClusterIP. The client then connects to that IP, and kube-proxy's rules on the node rewrite the destination to one of the endpoint Pod IPs.

```
curl nginx-service.staging
  → CoreDNS (search-list expansion) → ClusterIP
  → kube-proxy rules on the node → a Pod IP in staging
```

A side effect of `ndots:5`: external names like `example.com` also walk the search list first, producing extra failed lookups. This is a known source of DNS load and latency in busy clusters.

### Across namespaces

Observed with the same `nginx-service` and `nginx-deployment` in both `default` and `staging`: a Pod in `default` can reach the Service in `staging` by name. A Service only selects Pods in its own namespace, even though Pods in both carry `app=nginx`. See [namespace.md](namespace.md).

## Service types

| Type | Reachable from | What it adds |
|---|---|---|
| **ClusterIP** (default) | Inside the cluster only | A virtual IP and DNS name |
| **NodePort** | `<node-ip>:<nodePort>` from outside | A ClusterIP, plus the same port (30000–32767) opened on every node |
| **LoadBalancer** | External IP | A NodePort, plus a request to the cloud provider for an external load balancer |
| **ExternalName** | Inside the cluster | No proxying. DNS returns a CNAME to an external hostname |

```
LoadBalancer ⊃ NodePort ⊃ ClusterIP
```

- **ClusterIP** is for service-to-service traffic. Most Services are this.
- **NodePort** is mostly for tests, or as the building block for the others.
- **LoadBalancer** exposes one Service publicly. On EKS it provisions an AWS load balancer per Service, which costs money. For many HTTP services, one Ingress (Phase 3) in front of ClusterIP Services is the usual pattern.
- **ExternalName** gives an external dependency a cluster-internal name, so it can be swapped later without changing app config.
- **Headless** (`clusterIP: None`) has no virtual IP. DNS returns the Pod IPs directly. StatefulSets use it so each Pod gets a stable, individually addressable name (Phase 2).

### On kind

- The node is a Docker container, so a NodePort isn't reachable from the Mac without extra port mappings configured when the cluster is created.
- A `LoadBalancer` Service stays `<pending>`, because nothing exists to provision one. MetalLB or `cloud-provider-kind` can fill that gap.

## Reaching a Service

**From inside the cluster** (the intended use). Run a temporary Pod and call the Service by name:

```bash
kubectl run tmp --rm -it --restart=Never --image=curlimages/curl -- curl -s nginx-service
```

**From the Mac:**

```bash
kubectl port-forward svc/nginx-service 8080:80
curl localhost:8080
```

`port-forward` to a Service tunnels through the API server to **one** backing Pod. It works for a ClusterIP Service, but it does not exercise the Service's load balancing, so it is a connectivity check, not a test of the Service as a Service. The in-cluster call is the real test, because it goes through DNS and kube-proxy.

**From the node.** kube-proxy's rules live on the node, so the ClusterIP works there:

```bash
kubectl get svc nginx-service
docker exec learn-control-plane curl -s <CLUSTER-IP>   # if curl exists in the node image
```

## Behaviour that follows from the mechanism

- **Deleting a backing Pod:** the ReplicaSet creates a replacement. The Service's endpoints drop the old IP and add the new one once it is Ready. Clients that use the Service name never need to know.
- **Scaling to 0:** the Service still exists and keeps its ClusterIP, but it has no endpoints, so connections are refused.
- **Changing a Pod's label so it no longer matches the selector:** the Service stops sending traffic to it. The Deployment's ReplicaSet also no longer counts it as one of its Pods, so it releases the Pod and creates a replacement. The relabelled Pod keeps running, but is orphaned.
- **Selector typo or wrong labels:** the Service exists but `kubectl get endpoints <svc>` is empty. This is the most common Service bug.
- **Wrong `targetPort`:** endpoints are populated but connections fail, because the Pods aren't listening on that port.

## Commands

```bash
kubectl get svc
kubectl get svc nginx-service -o wide        # shows the selector
kubectl describe svc nginx-service           # selector, ports, endpoints
kubectl get endpoints nginx-service
kubectl get endpointslices
kubectl get pods --show-labels
kubectl get pods -l app=nginx                # the selector query, run by hand
```

## Debugging checklist

When a Service doesn't work, check in this order:

1. `kubectl get endpoints <svc>`. If it is empty, the selector doesn't match any Ready Pod. Compare `kubectl describe svc` with `kubectl get pods --show-labels`, and check that the Pods are Ready.
2. If endpoints exist, check `targetPort` against the port the container actually listens on.
3. Call the Pod IP directly from a temporary Pod. If that works but the Service doesn't, the problem is the Service. If it fails too, the problem is the app.
4. Check that the name resolves (`nslookup nginx-service` from a temporary Pod) and that you're using the right namespace in the name.
