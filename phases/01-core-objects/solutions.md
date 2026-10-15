# Phase 01 solutions: debug challenge

**Spoilers.** Open this only after you have diagnosed each case in `labs/01-core-objects/broken/`, or after you have been stuck for a long time.

## 01-wont-apply

**Symptom:** `kubectl apply` is rejected with `selector does not match template labels`.

**Root cause:** the Deployment's `spec.selector.matchLabels` is `app: web`, but the Pod template's labels are `app: webapp`. The Deployment couldn't find its own Pods, so the API refuses it.

**Fix:** make them identical.

## 02-service-unreachable-a

**Symptom:** Pods are `Running`, requests to the Service fail.

**How to find it:** `kubectl get endpoints shop-02 -n debug` shows `<none>`. Compare `kubectl describe svc shop-02 -n debug` (the selector) with `kubectl get pods -n debug --show-labels`.

**Root cause:** the Service selector is `app: shpo`, a typo of `app: shop`. It matches no Pods, so it has no endpoints.

**Fix:** correct the selector.

## 03-service-unreachable-b

**Symptom:** Pods are `Running`, the Service has endpoints, and requests through it fail.

**How to find it:** `kubectl get endpoints api-03 -n debug` shows the Pod IPs with port `8080`. `kubectl describe svc` shows `TargetPort: 8080`. nginx listens on 80.

**Root cause:** the Service's `targetPort` is 8080, but the container listens on 80. Traffic is sent to a port nothing is listening on.

**Fix:** set `targetPort: 80`.

## 04-keeps-restarting

**Symptom:** `CrashLoopBackOff` and a growing `RESTARTS` count.

**How to find it:** `kubectl logs deploy/worker-04 -n debug --previous` shows `fatal: DB_HOST is not set`. `describe pod` shows the container exiting with code 1.

**Root cause:** the container exits with an error when the `DB_HOST` environment variable isn't set, and the manifest doesn't set it. The restart policy restarts it, it fails again, and the kubelet backs off between attempts.

**Fix:** add `DB_HOST` under the container's `env`.

## 05-stuck-pending

**Symptom:** the Pod stays `Pending`.

**How to find it:** `kubectl describe pod -n debug -l app=batch` shows a `FailedScheduling` event: `Insufficient memory` (the node doesn't have enough Allocatable). Compare against `kubectl describe node` Allocatable.

**Root cause:** the Pod requests 500Gi of memory and 2 CPUs, more than the node can offer. The scheduler has no node that fits, so it never assigns one, and the Pod never reaches a kubelet.

**Fix:** request a realistic amount (for example `128Mi`).

## 06-service-unreachable-c

**Symptom:** Pods are `Running`, but the Service doesn't send traffic to them.

**How to find it:** the Service isn't in the `debug` namespace with the Pods. `kubectl get svc -A | grep site-06` shows it in `default`, and `kubectl get endpoints site-06 -n default` shows `<none>`.

**Root cause:** the Service's `metadata.namespace` is `default`, but the Pods are in `debug`. A Service selects only Pods in its own namespace.

**Fix:** put the Service in `debug`.
