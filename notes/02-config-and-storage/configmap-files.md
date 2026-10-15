# ConfigMap as files

Exercise 2. Manifests: [labs/02-config-and-storage/configmap-files/](../../labs/02-config-and-storage/configmap-files/)

A ConfigMap can be mounted into a container as a directory of files. Each **key is a file name** and each **value is the file's content**.

## Putting a file in a ConfigMap

A ConfigMap can't point at a file on disk. The content goes inside it:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: lab-configmap-file
data:
  index.html: |
    <html>...</html>
```

`|` starts a multi-line block. To have kubectl read an existing file instead:

```bash
kubectl create configmap web-content --from-file=index.html
kubectl create configmap web-content --from-file=index.html --dry-run=client -o yaml   # print the YAML only
```

- Key names may contain letters, digits, `-`, `_`, and `.`. A key such as `greeting` becomes a file called `greeting` with **no extension**. Kubernetes doesn't add one. To get `.txt`, name the key `greeting.txt`. The extension only matters to the program reading the file (nginx uses it for `Content-Type`).
- Several keys become several files in the same directory.
- Text goes under `data`, binary under `binaryData` (base64). A ConfigMap is limited to 1 MiB in total.

## Mounting it

Two parts in the Pod spec:

```yaml
containers:
  - name: nginx
    volumeMounts:
      - name: config-volume          # which volume
        mountPath: /usr/share/nginx/html   # where it appears in the container
volumes:
  - name: config-volume
    configMap:
      name: lab-configmap-file
```

Served at `localhost:8080/index.html` through `kubectl port-forward`.

## Whole-ConfigMap mount

Without `subPath`, the ConfigMap is mounted as a directory at `mountPath`.

Observed, with the whole ConfigMap mounted at `/usr/share/nginx/html`:

```
index.html -> ..data/index.html
..data -> ..2026_10_15_10_08_36.51475717
..2026_10_15_10_08_36.51475717/
```

A plain nginx image has `50x.html` and `index.html` in that directory. With the mount, only the ConfigMap's keys are there, and `curl localhost/50x.html` returned `404`.

- **The mount hides everything the image had at that path.** Only the ConfigMap's keys appear.
- **Updates work through a symlink swap.** `index.html` is a symlink to `..data/index.html`, and `..data` points at a timestamped directory. When the ConfigMap changes, the kubelet writes the new content into a new timestamped directory and atomically repoints `..data`. The paths an app opens stay valid, and it never sees a half-written file.
- The kubelet syncs periodically, so expect a delay of up to about a minute, not an instant change.

## `subPath`

`subPath` selects one key and bind-mounts it as a single file. `mountPath` must then be the **full path of the file**, not a directory:

```yaml
volumeMounts:
  - name: config-volume
    mountPath: /usr/share/nginx/html/index.html
    subPath: index.html
```

The Pod's `Mounts:` line in `kubectl describe pod` shows `(rw,path="index.html")` for a `subPath` mount.

It leaves the other files in the directory alone, which is what whole-directory mounting can't do.

### A `subPath` mount does not update

Tested by patching the ConfigMap and polling the file inside the running Pod for 150 seconds. It never changed, and the ConfigMap itself had changed. A `subPath` file is bind-mounted from the volume as it was when the container started, so the kubelet's later symlink swap doesn't reach it. The Kubernetes docs state the same.

To pick up a change, restart the Pod: `kubectl rollout restart deploy/<name>`.

Careful when testing this: a Pod that starts *after* you edit the ConfigMap shows the new content from the start. That looks like an update but isn't one.

### A wrong `mountPath` with `subPath`

First attempt used `mountPath: /usr/share/nginx/html` together with `subPath: index.html`, which asks for a **file** to be mounted onto a **directory**. The new Pod went to `RunContainerError`:

```
error mounting ".../volume-subpaths/config-volume/..." to rootfs at "/usr/share/nginx/html": ... not a directory
```

The rollout stalled the same way the bad-image rollout did in Phase 01: the Deployment kept the old Pod running, so `curl` kept working and showed the old content. Nothing seen through the old Pod said anything about the `subPath` change. Check `kubectl get pods` and `describe pod` for the new Pod, and don't trust the old one.

Fix: `mountPath: /usr/share/nginx/html/index.html`.

## When to use which

| Need | Use |
|---|---|
| Config that updates without a restart | Whole-ConfigMap mount, into a directory you own |
| Add one file to a directory that already has other files | `subPath`, and restart to change it |
| A value for the app's environment | `env` / `envFrom` (see [configmap-env.md](configmap-env.md)), and restart to change it |

Even a whole-directory mount only updates the **file**. An app that reads its config once at startup won't notice. nginx re-reads static files like `index.html` on every request, but not its own config file, which needs a reload.

## Commands

```bash
kubectl exec deploy/<name> -- ls -la /usr/share/nginx/html
kubectl exec deploy/<name> -- cat /usr/share/nginx/html/index.html
kubectl describe pod <pod>              # the Mounts: section
kubectl rollout status deploy/<name>
```
