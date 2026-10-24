# Roadmap

Check off a phase only when you can do its exercises without the docs open.

- [x] **01. Core objects** — Pod, ReplicaSet, Deployment, Service, Namespace, labels and selectors, `kubectl` ([phase](phases/01-core-objects/README.md), [notes](notes/01-core-objects/))
- [x] **02. Config and storage** — ConfigMap, Secret, volumes, PV/PVC, StatefulSet ([phase](phases/02-config-and-storage/README.md), [notes](notes/02-config-and-storage/))
- [ ] **03. Networking** — Service types, Ingress, cluster DNS, NetworkPolicy
- [ ] **04. Operations** — probes, requests/limits, HPA, rolling updates, RBAC, Jobs and CronJobs
- [ ] **05. Packaging and cloud** — Helm, Kustomize, EKS with Terraform, IRSA, ingress controller
- [ ] **06. Internals** — API server, etcd, scheduler, controllers, kubelet, CNI/CSI/CRI, CRDs and operators
- [ ] **07. Observability and GitOps** — Prometheus, Grafana, logging, ArgoCD
- [ ] **08. AI workloads** — GPU scheduling, model serving, KEDA, batch and training jobs

## Projects

- [ ] 1. Containerize a web service and deploy it with probes and self-healing
- [ ] 2. Add Redis and MongoDB with persistent storage
- [ ] 3. Ingress plus autoscaling under load
- [ ] 4. RAG/chatbot stack packaged as a Helm chart
- [ ] 5. Serve a small LLM behind the API with autoscaling
- [ ] 6. EKS via Terraform with GitHub Actions and ArgoCD
- [ ] 7. A small operator with a CRD

Phase folders are added as the course reaches them.
