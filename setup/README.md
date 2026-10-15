# Setup

## Tools

| Tool | Purpose | Check |
|---|---|---|
| Docker (Docker Desktop or OrbStack) | Runs the local cluster nodes as containers | `docker info` |
| kubectl | Talks to the cluster | `kubectl version --client` |
| kind | Creates local clusters | `kind version` |

Install via your package manager (on macOS, `brew install kubectl kind`).

## Create a cluster

```bash
kind create cluster --name learn
kubectl cluster-info --context kind-learn
kubectl get nodes
```

## Verify

```bash
kubectl get pods -A
```

You should see system pods in the `kube-system` namespace. Before moving on, be able to say what each one does.

## Reset

```bash
kind delete cluster --name learn
```

Deleting and recreating the cluster takes under a minute. Do it whenever state gets confusing.

## Habits worth setting up now

```bash
alias k=kubectl
source <(kubectl completion zsh)   # or bash
```
