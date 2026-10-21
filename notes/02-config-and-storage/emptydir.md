# emptyDir

Exercise 4. Manifests: [labs/02-config-and-storage/emptydir/](../../labs/02-config-and-storage/emptydir/)

An `emptyDir` is an empty directory that Kubernetes creates when a Pod starts on a node and deletes when the Pod goes away. Every container in the Pod can mount it, so it works as scratch space shared between containers.

```yaml
volumes:
  - name: shared-volume
    emptyDir: {}
```

Each container lists it in `volumeMounts`, at the same path or different ones. There's no Kubernetes API for writing to it. Containers read and write files under the `mountPath` like any directory.

## Sharing between containers

Container 1 wrote a file and container 2 read it from the same path:

```yaml
- name: emptydir-container-1
  command: ["sh", "-c", "echo Hello from the emptyDir volume! >> /usr/shared/message.txt && sleep 3600"]
  volumeMounts:
    - name: shared-volume
      mountPath: /usr/shared
- name: emptydir-container-2
  command: ["sh", "-c", "cat /usr/shared/message.txt && sleep 3600"]
  volumeMounts:
    - name: shared-volume
      mountPath: /usr/shared
```

```bash
kubectl exec emptydir-pod -c emptydir-container-2 -- cat /usr/shared/message.txt
# Hello from the emptyDir volume!
```

- `exec` and `logs` need `-c <container>` in a multi-container Pod.
- `>>` appends and `>` overwrites. A container restart re-runs its command, so `>>` adds another line each time.
- Containers in a Pod start in order but don't wait for each other. Container 2's `cat` reads the file with no check that container 1 has written it yet. It worked here, but the order is not guaranteed.

## Lifetime

| Event | Data in the `emptyDir` |
|---|---|
| A container crashes or is killed and restarts | Survives |
| The Pod is deleted and recreated | Gone |

- Killed container 1 with `kubectl exec emptydir-pod -c emptydir-container-1 -- kill 1`. The container restarted and the file was still there.
- Deleted the Pod and applied it again. `message.txt` had **one** line. That line wasn't surviving data. The new Pod got a new empty directory, and container 1 wrote its line when it started. Earlier restarts had appended more lines, and none of those were there.

The volume belongs to the **Pod**, not to a container. The container's own filesystem is lost on every container restart, and an `emptyDir` is not.

A Deployment's replacement Pod also starts with an empty directory. Data that must outlive the Pod needs a PVC (Exercise 5).

## Options

```yaml
volumes:
  - name: shared-volume
    emptyDir:
      medium: Memory
      sizeLimit: 64Mi
```

- **Default medium:** the node's disk, which counts against the node's ephemeral storage.
- **`medium: Memory`:** a tmpfs, so RAM-backed and faster. It counts against the container's memory limit.
- **`sizeLimit`:** caps the size. With the default medium, a Pod that exceeds it is evicted. This is from the Kubernetes docs and wasn't tested here.

Observed with `medium: Memory` and `sizeLimit: 64Mi`:

```bash
kubectl exec emptydir-pod -c emptydir-container-1 -- df -h /usr/shared
```

```
Filesystem                Size      Used Available Use% Mounted on
tmpfs                    64.0M      4.0K     64.0M   0% /usr/shared
```

The volume is a `tmpfs` capped at the `sizeLimit`, and the 4 KiB used is `message.txt`. Writing past the limit was not tested.

### Changing the volume of a running Pod

Adding `medium` and `sizeLimit` to the existing Pod and applying it was rejected:

```
The Pod "emptydir-pod" is invalid: spec: Forbidden: pod updates may not change fields other than ...
```

Volumes can't change on a live Pod (same rule as in [configmap-env.md](configmap-env.md)). Delete and recreate the Pod, or use `kubectl replace --force -f pod.yaml`.

### `df`

`df -h <path>` reports the size, usage, and free space of the filesystem that contains `<path>`. `-h` prints sizes in readable units. It shows the `Filesystem` column but not the filesystem type, which `mount | grep <path>` shows.

## When to use it

- Sharing files between containers, such as a sidecar that ships the app's log files, or an init container that prepares files for the main container.
- Scratch or cache space that's fine to lose and rebuild.
- Not for data that must survive the Pod being deleted or rescheduled.

## Commands

```bash
kubectl exec <pod> -c <container> -- cat /usr/shared/<file>
kubectl exec <pod> -c <container> -- kill 1
kubectl exec <pod> -c <container> -- df -h /usr/shared
kubectl exec <pod> -c <container> -- mount | grep /usr/shared
kubectl get pod <pod>
kubectl describe pod <pod>
```
