# Phase 01: Core objects

## Goals

By the end you can:

- Explain the declarative model: you declare desired state, controllers reconcile toward it.
- Create, inspect, and debug Pods, Deployments, and Services using manifests you wrote.
- Explain how labels and selectors connect objects.
- Debug a failing workload using `describe`, `logs`, and `events`.

## Concepts

- **Cluster anatomy:** control plane vs. worker nodes, and the `kube-system` components.
- **Pod:** the smallest deployable unit. One or more containers sharing network and storage.
- **ReplicaSet:** keeps N identical Pods running.
- **Deployment:** manages ReplicaSets and gives rolling updates and rollback.
- **Service:** a stable address in front of a changing set of Pods.
- **Namespace:** a scope for names and policies.
- **Labels and selectors:** how objects find each other.
- **Reconciliation loop:** observe, diff against desired state, act, repeat.

## Tools

- `kubectl get | describe | logs | exec | apply | delete | explain`
- `kubectl get -o yaml` to see what the cluster actually stored

## Reading

- Kubernetes docs: Concepts > Overview, Workloads, Services
- `kubectl explain pod.spec` (use it constantly)

## Files

- [exercises.md](exercises.md): tasks and questions
- [hints.md](hints.md): open only when stuck
