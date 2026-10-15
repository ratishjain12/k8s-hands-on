# Phase 01 hints

Try for at least 10 minutes before opening a hint. Hints are ordered from nudge to near-answer.

## 1. A bare Pod

1. `kubectl explain pod` shows the required top-level fields.
2. There are four top-level fields you need: `apiVersion`, `kind`, `metadata`, `spec`.
3. For access without a Service, look at `kubectl port-forward`.

## 2. Deployment

1. The Deployment's `selector` must match the Pod template's labels. If they don't, the API rejects it.
2. Imperative scaling: `kubectl scale`. Declarative: change `replicas` and re-apply.

## 3. Rolling update

1. `kubectl rollout status`, `history`, and `undo` are the three commands you need.
2. A failed new Pod doesn't take down the old ones. Ask why: look at the rolling update strategy defaults (`maxUnavailable`, `maxSurge`).
3. `kubectl describe pod` and its Events section show pull errors.

## 4. Services

1. A Service finds Pods through `spec.selector`. `kubectl get endpoints <svc>` shows what it matched.
2. To test from inside the cluster, run a temporary Pod with a shell or curl image and call the Service name.
3. From your machine: `port-forward` is one way. The other involves a Service type or a node port. Check `kind` docs on why NodePort isn't directly reachable on macOS.

## 5. Namespaces

1. Cross-namespace DNS format: `<service>.<namespace>.svc.cluster.local`.
