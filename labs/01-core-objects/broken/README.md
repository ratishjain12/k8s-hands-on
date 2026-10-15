# Debug challenge

Six broken workloads. Each one has a different root cause. Find it using `kubectl` output only, fix the manifest, and confirm the fix works.

These manifests are intentionally broken. If you fix them in your own clone, don't commit the fixes back.

## Rules

1. Don't read the manifests first. Apply one, look at what the cluster says, and diagnose from there.
2. Spend 10 minutes before you ask for help or open the hints.
3. Fix the file, then re-apply it. Don't patch the live objects with `kubectl edit`.
4. For each case, write an entry in `notes/01-core-objects/debug.md` with the symptom, the commands you ran, the root cause, and the fix.

## Setup

```bash
kubectl apply -f 00-namespace.yaml
```

Everything lives in the `debug` namespace. Apply the cases one at a time:

```bash
kubectl apply -f 01-wont-apply.yaml
```

## Cases

| File | Symptom |
|---|---|
| `01-wont-apply.yaml` | `kubectl apply` is rejected |
| `02-service-unreachable-a.yaml` | The Pods are running, but nothing answers through the Service |
| `03-service-unreachable-b.yaml` | The Pods are running and the Service lists endpoints, but requests through the Service fail |
| `04-keeps-restarting.yaml` | The Pod starts, then keeps restarting |
| `05-stuck-pending.yaml` | The Pod never starts |
| `06-service-unreachable-c.yaml` | The Pods are running, but the Service doesn't send traffic to them |

To test a Service from inside the cluster:

```bash
kubectl run tmp -n debug --rm -it --restart=Never --image=curlimages/curl -- curl -s -m 5 <service>
```

## Cleanup

```bash
kubectl delete namespace debug
kubectl delete svc site-06 -n default
```

## Done when

All six work, and `notes/01-core-objects/debug.md` has an entry for each one with the commands that revealed the cause.
