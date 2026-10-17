# Secrets

Exercise 3. Manifests: [labs/02-config-and-storage/secret/](../../labs/02-config-and-storage/secret/)

A Secret is shaped like a ConfigMap (key-value pairs, used as env vars or files), but it's meant for sensitive values and is handled a little differently.

## `data` vs `stringData`

| Field | You write | Stored as |
|---|---|---|
| `data` | base64-encoded values | `data`, as written |
| `stringData` | plain text | merged into `data`, base64-encoded by the API server |

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: lab-secret-string-data
type: Opaque
stringData:
  username: admin
  password: mypassword
```

- `data` values must already be valid base64. `admin` is not (5 characters), and the API server rejects it. Encode with `echo -n 'admin' | base64`. `-n` matters because without it the newline is encoded too, and it becomes part of the value.
- `stringData` is write-only. Reading the Secret back returns only `data`.
- One file with both fields is legal. Keys from both are merged, and for a key in both, `stringData` wins. This is documented behavior and was not tested here. Use separate Secrets when comparing the two forms.

## What each command shows

- `kubectl get secret <name> -o yaml`: the values, base64-encoded.
- `echo '<value>' | base64 -d`: the plain value.
- `kubectl describe secret <name>`: the type and the **size in bytes** of each key (`password: 10 bytes`, `username: 5 bytes`). It never prints the values.

`kubectl apply` stores the manifest you applied in the `kubectl.kubernetes.io/last-applied-configuration` annotation. For a `stringData` Secret, that annotation holds the **plaintext** values:

```
"stringData":{"password":"mypassword","username":"admin"}
```

So `get -o yaml` shows them even though `stringData` is not a real field of the stored object. `kubectl create secret` or `kubectl apply --server-side` doesn't write this annotation.

## Is base64 encryption?

No. It's an encoding, and anyone can reverse it with `base64 -d`, no key needed.

- **Where it's stored:** etcd, as base64 by default. Encryption at rest is a separate setting on the API server.
- **Who can read it:** anyone with `get` on Secrets in that namespace, and anyone who can create a Pod there and mount it. In this cluster, whoever has kubectl access can.

## Consuming a Secret

| Form | Field path |
|---|---|
| Single key as env var | `env[].valueFrom.secretKeyRef` (`name`, `key`) |
| All keys as env vars | `envFrom[].secretRef` |
| Files | `volumes[].secret.secretName`, plus a `volumeMounts` entry |

```yaml
env:
  - name: SECRET_USERNAME
    valueFrom:
      secretKeyRef:
        name: lab-secret-data
        key: username
```

```yaml
volumeMounts:
  - name: secret-volume
    mountPath: /etc/secrets
    readOnly: true
volumes:
  - name: secret-volume
    secret:
      secretName: lab-secret-data
```

Compared with a ConfigMap volume: `configMap:` becomes `secret:`, and `name:` becomes `secretName:`. The rest is the same.

Behavior matches ConfigMaps:

| Consumed as | After the Secret changes |
|---|---|
| env var | No update until the container restarts (`kubectl rollout restart deploy/<name>`) |
| whole-directory mount | Updates through the `..data` symlink swap, with a delay |
| `subPath` mount | No update until the container restarts |

## Mounted files

Observed in the Pod, mounted at `/etc/secrets`:

```
password -> ..data/password
username -> ..data/username
..data -> ..2026_10_17_15_43_04.4072889548
```

```
tmpfs on /etc/secrets type tmpfs (ro,relatime,size=8196880k,noswap)
```

- Same symlink layout as the ConfigMap mount. Each key is a file named after the key, with no extension, and the content is the **decoded** value.
- The mount is `tmpfs`, which is memory-backed. The values aren't written to the node's disk, and they disappear when the Pod is deleted. `tmpfs` is not a `/tmp` directory, and the files are at your `mountPath`.
- The Pod's ServiceAccount token is mounted the same way, at `/run/secrets/kubernetes.io/serviceaccount`.
- `defaultMode` sets file permissions, and `items` selects keys or renames files.

An app reads a mounted Secret as an ordinary file. Strip any trailing whitespace, and take the path from config (for example `DB_PASSWORD_FILE`) rather than hardcoding it.

## Production practice

Not run in this lab. This is how Secrets are normally handled:

- The source of truth is a secret manager (AWS Secrets Manager, Vault). A controller such as External Secrets Operator syncs it into a Kubernetes Secret, and Git holds only a reference (an `ExternalSecret`), never the value. The Secrets Store CSI driver can mount the value without creating a Secret object.
- etcd encryption at rest, tight RBAC on Secrets, and audit logging.
- Files are preferred over env vars for sensitive values, because env vars leak more easily (logs, crash dumps, child processes).
- Where possible, cloud identity (IRSA or Pod Identity on EKS) replaces stored credentials altogether.
- Rotation needs a restart for env vars. Tools like Reloader or a hash of the Secret in the Pod template trigger it automatically.

## Commands

```bash
echo -n 'admin' | base64
echo 'YWRtaW4=' | base64 -d
kubectl get secret <name> -o yaml
kubectl describe secret <name>
kubectl exec <pod> -- ls -la /etc/secrets
kubectl exec <pod> -- mount | grep secrets
kubectl exec deploy/<name> -- env | grep SECRET_
```
