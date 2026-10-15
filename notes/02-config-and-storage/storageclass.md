# StorageClass and dynamic provisioning

Exercise 0. Explored with `kubectl get storageclass`, `kubectl describe storageclass`, `kubectl get pv,pvc -A`, and `kubectl get pods -n local-path-storage`.

## The default StorageClass on kind

| Field | Value |
|---|---|
| Name | `standard` (the default) |
| provisioner | `rancher.io/local-path` |
| reclaimPolicy | `Delete` |
| volumeBindingMode | `WaitForFirstConsumer` |

- **provisioner:** the component that creates a PV when a PVC asks for this class. Here it carves out a directory on the node's disk.
- **reclaimPolicy `Delete`:** deleting the PVC also deletes the PV and its data. `Retain` would keep them.
- **volumeBindingMode `WaitForFirstConsumer`:** no PV is created until a Pod that uses the PVC is scheduled. Local storage lives on one specific node, so the system must know which node the Pod lands on before it decides where to create the volume. The alternative, `Immediate`, provisions as soon as the PVC exists.

## Dynamic provisioning

`kubectl get pv,pvc -A` returned nothing. PVs don't exist ahead of time. They are created on demand when a claim shows up.

The `local-path-provisioner` Pod in the `local-path-storage` namespace is a **controller**. It watches for PVCs that use the `standard` class and, for each one:

1. Creates a directory on the node's disk.
2. Creates a PV object pointing at that directory.
3. Binds the PV to the claim.

It follows the same pattern as the controllers in [../01-core-objects/cluster-components.md](../01-core-objects/cluster-components.md): watch for a desired state (a PVC), then act to make it real.

## Commands

```bash
kubectl get storageclass
kubectl describe storageclass standard
kubectl get pv,pvc -A
kubectl get pods -n local-path-storage
```
