# Namespace

Exercise 5. Manifest: [labs/01-core-objects/namespace/namespace.yaml](../../labs/01-core-objects/namespace/namespace.yaml)

A namespace is a **scope for names** inside one cluster. It is also the unit that access rules (RBAC), resource quotas, and network policies attach to, which is why teams and environments are usually split by namespace.

## Manifest

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: staging
```

No `spec` is needed. A Namespace is not itself namespaced. Labels are common, because policies and tooling select namespaces by label (Pod Security Admission uses them).

## Putting objects in a namespace

Two ways:

```bash
kubectl apply -f deployment.yaml -n staging     # flag at apply time
```

```yaml
metadata:
  name: nginx-deployment
  namespace: staging                            # fixed in the manifest
```

| | `-n` flag | `metadata.namespace` |
|---|---|---|
| Same file reused across namespaces | Yes | No, the file is tied to one |
| Risk | Forget the flag and the objects land in whatever namespace the current context points at, usually `default` | Applying the file always targets the right namespace |

Here the same `deployment.yaml` and `service.yaml` were applied to both `default` and `staging` with `-n staging`, and neither file contains a namespace.

## Observed

The same manifests in `default` and `staging` produced separate objects:

| | `default` | `staging` |
|---|---|---|
| Deployment | `nginx-deployment` 3/3 | `nginx-deployment` 3/3 |
| Service ClusterIP | `10.96.163.160` | `10.96.233.9` |
| Endpoints | the `default` Pods | `10.244.0.14`, `.15`, `.16` |

- A name only has to be unique **within** a namespace.
- Pods in both carry `app=nginx`, yet each Service's endpoints contain only the Pods in its own namespace. Selectors are scoped to the Service's namespace.
- A Pod in `default` could reach the `staging` Service and Pods. See the DNS section in [service.md](service.md).

## Not isolation

Namespaces give scoping, not isolation.

Separated by default:
- Names.
- Selectors.
- Things attached per namespace: RBAC, quotas, policies.

**Not** separated by default:
- **Network.** All Pods share one flat network. A Pod in one namespace can reach any Pod in another, by Pod IP or by Service DNS name.
- **Nodes.** Pods from every namespace share nodes and kernel, so one namespace can starve another of CPU and memory.
- **Cluster-scoped objects.** Nodes, PersistentVolumes, StorageClasses, and CRDs are not in any namespace. `kubectl api-resources --namespaced=false` lists them.
- **Permissions**, until RBAC is configured.

What adds isolation:

| Concern | Mechanism |
|---|---|
| Network traffic | NetworkPolicy (Phase 3) |
| Who can do what | RBAC Roles and RoleBindings (Phase 4) |
| Resource hogging | ResourceQuota and LimitRange |
| Dangerous Pods | Pod Security Admission labels |

For hard isolation between parties that don't trust each other, use separate clusters (or virtual clusters such as vcluster). Namespaces suit teams, environments, or apps that trust each other enough to share a cluster.

## Commands

```bash
kubectl get ns
kubectl get pods                     # current context's namespace only
kubectl get pods -n staging
kubectl get pods -A                  # all namespaces
kubectl config set-context --current --namespace=staging   # change the default namespace
kubectl config view --minify | grep namespace              # which namespace is active
```

Setting the default namespace is convenient but dangerous: a later `kubectl delete` or `apply` runs against whatever namespace the context is set to, with no prompt. Always check the namespace before destructive commands, or pass `-n` explicitly.

Deleting a namespace deletes every namespaced object inside it.

Built-in namespaces: `default`, `kube-system` (control plane and add-ons), `kube-public`, `kube-node-lease` (node heartbeats), plus `local-path-storage` on kind.
