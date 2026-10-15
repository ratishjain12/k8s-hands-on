# ConfigMap as environment variables

Exercise 1. Manifests: [labs/02-config-and-storage/configmap-env/](../../labs/02-config-and-storage/configmap-env/)

A ConfigMap holds non-sensitive configuration as key-value pairs, separate from the container image. The same image can then run with different settings in each environment.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: lab-configmap
data:
  DB_HOST: "localhost"
  DB_PORT: "5432"
```

## Two ways to consume it

| Form | Field path | Effect |
|---|---|---|
| Bulk | `envFrom[].configMapRef` | Every key becomes a variable with the same name |
| Single key | `env[].valueFrom.configMapKeyRef` | One key becomes one variable, and you pick the variable's name |

```yaml
# bulk (deployment.yaml)
envFrom:
  - configMapRef:
      name: lab-configmap

# single key (pod.yaml)
env:
  - name: DB_HOST
    valueFrom:
      configMapKeyRef:
        name: lab-configmap
        key: DB_HOST
```

- `envFrom` takes whole objects, so `configMapKeyRef` there is rejected: `unknown field "spec.containers[0].envFrom[0].configMapKeyRef"`. This kind of strict decoding error means the YAML is well-formed but a field sits in the wrong place. The path in the message shows where.
- If both forms define the same variable name, `env` wins over `envFrom`.
- If the named ConfigMap or key doesn't exist, the Pod gets stuck in `CreateContainerConfigError`.

Check the result inside the container:

```bash
kubectl exec <pod> -- env
kubectl exec deploy/lab-deployment -- env | grep DB_
```

`kubectl describe pod` shows where each value comes from, but not the resolved values for `envFrom`. `exec env` shows what the container actually has.

## Variables you didn't write

`env` also showed `KUBERNETES_*` and `NGINX_SERVICE_*` variables. These are **Service environment variables**: the kubelet injects them for every Service in the Pod's namespace that exists when the container starts. The name is the Service name uppercased with dashes turned into underscores, plus `_SERVICE_HOST`, `_SERVICE_PORT`, and `_PORT_<port>_TCP_*`.

- Services created after the Pod starts don't appear, so DNS is the reliable way to find a Service.
- A namespace with many Services puts many variables in every Pod. `enableServiceLinks: false` in the Pod spec turns this off.

## A changed ConfigMap doesn't reach a running container

Changed `DB_HOST` from `localhost` to `db.example.com` and applied the ConfigMap.

Observed:

- `kubectl get cm lab-configmap -o yaml` showed the new value.
- The running Pods (both the bare Pod and the Deployment's Pod) still had the **old** value.

The container's environment is built once, when the container starts. Nothing watches ConfigMaps and restarts Pods.

## Picking up the change

- **Deployment:** `kubectl rollout restart deploy/lab-deployment`. It adds a `kubectl.kubernetes.io/restartedAt` annotation to the Pod template. That's a template change, so the Deployment creates a new ReplicaSet and does a normal rolling update, and the new Pod gets a new name. It's better than `kubectl delete pod` because the new Pod must be Ready before the old one goes, so there's no gap with several replicas.
- **Bare Pod:** recreate it (`kubectl delete pod`, then apply, or `kubectl replace --force -f pod.yaml`).

After the restart, both showed `DB_HOST=db.example.com`.

## Pod spec is immutable

Changing a running Pod's environment in place is rejected:

```
The Pod "lab-pod" is invalid: spec: Forbidden: pod updates may not change fields other than
`spec.containers[*].image`, `spec.initContainers[*].image`, `spec.activeDeadlineSeconds`,
`spec.tolerations` (only additions to existing tolerations), `spec.terminationGracePeriodSeconds` ...
```

Only those few fields can change on a live Pod. Everything else means recreating it. A Deployment gets around this by never editing a Pod: it creates new Pods from a changed template and retires the old ones.
