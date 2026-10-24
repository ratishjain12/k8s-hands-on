# Debug challenge

Six broken workloads, one for each lab in this phase. Find the root cause using `kubectl` output only, fix the manifest, and confirm the fix works.

These manifests are intentionally broken. If you fix them in your own clone, don't commit the fixes back.

## Rules

1. Don't read the manifests first. Apply one, look at what the cluster says, and diagnose from there.
2. Spend 10 minutes before you ask for help or open the hints.
3. Fix the file, then re-apply it. Don't patch the live objects with `kubectl edit`.
4. For each case, write an entry in `notes/02-config-and-storage/debug.md` with the symptom, the commands you ran, the root cause, and the fix.

## Setup

```bash
kubectl apply -f 00-namespace.yaml
```

Everything lives in the `debug-02` namespace. Apply the cases one at a time:

```bash
kubectl apply -f 01-configmap-env.yaml
```

## Cases

| File | Lab it comes from | Symptom |
|---|---|---|
| `01-configmap-env.yaml` | ConfigMap as env | The Pod never starts |
| `02-configmap-file.yaml` | ConfigMap as files | The Pod starts, fails, and restarts |
| `03-secret-volume.yaml` | Secrets | `kubectl apply` is rejected |
| `04-emptydir.yaml` | emptyDir | `kubectl apply` is rejected |
| `05-pvc.yaml` | PVC | The Pod and its claim never leave `Pending` |
| `06-statefulset.yaml` | StatefulSet | Every Pod is `Running`, but the Service has no endpoints and Pod names don't resolve |

To test DNS from inside the cluster:

```bash
kubectl run tmp -n debug-02 --rm -it --restart=Never --image=busybox:1.36 -- ping -c1 <name>
```

## Cleanup

```bash
kubectl delete namespace debug-02
```

Deleting the namespace also deletes the claims and their volumes.

## Done when

All six work, and `notes/02-config-and-storage/debug.md` has an entry for each one with the commands that revealed the cause.
