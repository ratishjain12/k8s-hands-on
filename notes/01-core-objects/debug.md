# Debug log

Cases from [labs/01-core-objects/broken/](../../labs/01-core-objects/broken/README.md). One entry per case: symptom, commands, root cause, fix.

The manifests in `broken/` are kept broken on purpose so others can work through them. The fixes live only in this log.

## 01: won't apply

**Symptom:** `kubectl apply -f 01-wont-apply.yaml` failed. Two different errors appeared, one after the other.

**Error 1:**

```
Error from server (NotFound): ... namespaces "debug" not found
```

- Root cause: the manifest sets `metadata.namespace: debug`, but the namespace didn't exist yet. A namespaced object needs its namespace to exist first.
- Fix: `kubectl apply -f 00-namespace.yaml`.

**Error 2:**

```
The Deployment "web-01" is invalid: spec.template.metadata.labels: Invalid value: {"app":"webapp"}: `selector` does not match template `labels`
```

- Root cause: `spec.selector.matchLabels` was `app: web`, but the Pod template's labels were `app: webapp`. The Deployment couldn't match its own Pods, so the API rejected it.
- Fix: make them identical (`app: webapp` in both).

**Takeaways:**

- These two errors come from different layers. The namespace error is a missing dependency. The selector error is a validation rule on the object itself, and it fires even in a namespace that exists.
- The selector must match the template labels. It is also immutable after creation, so a mistake found later means recreating the Deployment.
- When `apply` is rejected, nothing is created. The error message names the exact field that failed validation.

## 02: service unreachable (a)

**Symptom:** the Deployment's Pods were `Running` and `Ready`, but a `curl` to the Service from a temporary Pod failed ("service unreachable").

**Root cause:** the Service's `spec.selector` didn't match the labels on the Pods, so the Service had no backends.

**Fix:** correct the Service selector so it matches the Pod template labels (`app: shop`), then re-apply the file.

**How to confirm:**

```bash
kubectl get endpoints shop-02 -n debug                  # <none> means the selector matched no Ready Pod
kubectl describe svc shop-02 -n debug                   # the selector the Service is using
kubectl get pods -n debug --show-labels                 # the labels the Pods actually have
```

**Takeaways:**

- A Service's health is independent of its Pods' health. The Pods can be perfectly healthy while the Service points at nothing, because the only link between them is the label selector.
- `curl` from a temporary Pod is the real test of a Service, because it goes through DNS and kube-proxy. It tells you the Service is broken, but not why. Empty endpoints tell you why.
- First check for any Service problem: does `kubectl get endpoints <svc>` list addresses?

## 03: service unreachable (b)

**Symptom:** Pods `Running`, the Service had endpoints, but requests through the Service failed.

**Root cause:** the Service's `targetPort` was `8080`, but the container (nginx) listens on `80`. The selector was fine, so the endpoints existed, but traffic was sent to a port nothing was listening on.

**Fix:** set `targetPort: 80` and re-apply.

**How to confirm:**

```bash
kubectl get endpoints api-03 -n debug      # the endpoint addresses end in :8080, not the port the app uses
kubectl describe svc api-03 -n debug       # TargetPort: 8080
```

**Takeaways:**

- Empty endpoints mean a selector problem (case 02). Populated endpoints with a failing connection point to the port: look at the `:port` on each endpoint address.
- `targetPort` must be the port the container actually listens on. `containerPort` in the Pod spec is informational and wouldn't have caught this, because it doesn't enforce anything.
- Calling a Pod IP directly on its real port works while the Service doesn't. That comparison isolates the fault to the Service.

## 04: keeps restarting

**Symptom:** the Pod started, then kept restarting.

**Diagnosis:** the Pod's logs showed `fatal: DB_HOST is not set`.

**Root cause:** the container exits with code 1 when the `DB_HOST` environment variable isn't set, and the manifest didn't set it. The restart policy restarts the container, it fails again, and the kubelet backs off longer between each attempt (`CrashLoopBackOff`).

**Fix:** add the variable under the container in the Deployment and re-apply:

```yaml
env:
  - name: DB_HOST
    value: "db.example.com"
```

After the re-apply the new Pod was `1/1 Running` with `RESTARTS 0`.

**Commands for a crashing Pod:**

```bash
kubectl get pods -n debug                          # STATUS CrashLoopBackOff, RESTARTS climbing
kubectl logs <pod> -n debug                        # current container's output
kubectl logs <pod> -n debug --previous             # the container that just crashed
kubectl describe pod <pod> -n debug                # Last State, Exit Code, Events
```

**Takeaways:**

- `CrashLoopBackOff` is a symptom, not a cause. The cause is in the app's own output, so the logs come first.
- The current container may be only seconds old and have printed nothing yet, so `--previous` is often the one that shows the crash.
- `logs` is the wrong tool when the container never started (`ImagePullBackOff`, `Pending`). Use `describe` and Events there.
- Changing the Pod template made the Deployment start a new Pod through a rolling update. The old crashing one was replaced, not repaired.

## 05: stuck pending

**Symptom:** the Pod stayed `Pending` and never started.

**Diagnosis:** `kubectl describe pod` showed a `FailedScheduling` event from `default-scheduler`:

```
0/1 nodes are available: 1 Insufficient memory. preemption: 0/1 nodes are available: 1 Preemption is not helpful for scheduling.
```

**Root cause:** the container requested `cpu: "2"` and `memory: 500Gi`. The node's **Allocatable** is 8 CPUs and about 7.8Gi of memory (`8196880Ki`). The CPU request fits, the memory request doesn't, so no node qualifies. The scheduler never assigns a node, so the Pod never reaches a kubelet and no container is ever created.

**Fix:** lower the requests to something realistic and re-apply: `cpu: 100m` and `memory: 128Mi`.

**Commands:**

```bash
kubectl get pods -n debug                          # STATUS Pending
kubectl describe pod <pod> -n debug                # Events: FailedScheduling and the reason
kubectl describe node learn-control-plane          # Allocatable, and what is already requested
```

**Takeaways:**

- `Pending` with a `FailedScheduling` event means the scheduler couldn't place the Pod. Nothing ran, so `kubectl logs` has nothing to show. The reason is always in the Events.
- The scheduler compares a Pod's requests against the node's **Allocatable**, minus what other Pods have already requested. It doesn't look at Capacity, and it doesn't look at actual usage.
- Requests are what the scheduler reserves. A Pod that requests more than any node has stays `Pending` forever. The Deployment is fine in the meantime, and the replicas it can't place just wait.
- The same message with another resource name (`Insufficient cpu`) means the cpu request is the problem.

## 06: service unreachable (c)

**Symptom:** the Pods were `Running`, but the Service didn't send traffic to them.

**Diagnosis:** the Service existed, but not in the Pods' namespace:

```bash
kubectl get svc -A | grep site-06          # the Service was in default
kubectl get endpoints site-06 -n default   # <none>
```

**Root cause:** the Service had `metadata.namespace: default`, while the Deployment's Pods were in `debug`. A Service selects only Pods in its own namespace, so the selector (`app: site`, which was correct) had nothing to match.

**Fix:** set the Service's `metadata.namespace` to `debug`, re-apply, and delete the old Service. Apply creates a new Service in the new namespace and doesn't move the old one:

```bash
kubectl apply -f 06-service-unreachable-c.yaml
kubectl delete svc site-06 -n default
kubectl get endpoints site-06 -n debug     # now lists the two Pod IPs (10.244.0.33:80, 10.244.0.34:80)
```

**Takeaways:**

- Empty endpoints with a correct-looking selector: check the namespace. `kubectl get svc -A` is how to find a Service when you don't know where it is.
- Nothing warns you about this. Both objects are valid on their own. The only link between them is the label selector, and it is scoped to the namespace.
- The `-n` flag decides where an object lands, and `apply` doesn't move objects between namespaces. Changing the namespace in the file creates a second object and leaves the first one behind.
