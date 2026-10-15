# Phase 02 exercises

Write every manifest by hand in `labs/02-config-and-storage/<exercise>/`. For each exercise, write your prediction down before running anything. Put your notes in `notes/02-config-and-storage/`.

## 0. Explore storage in your cluster

1. Run `kubectl get storageclass` and `kubectl describe storageclass`. Which one is the default? What are its provisioner, `reclaimPolicy`, and `volumeBindingMode`?
2. Run `kubectl get pv,pvc -A`. What exists already?
3. Run `kubectl get pods -n local-path-storage`. What is this Pod's job, based on the StorageClass's provisioner?

## 1. ConfigMap as environment variables

1. Write a ConfigMap with a few keys (for example `APP_MODE` and `LOG_LEVEL`).
2. Write a Pod that receives one key through a single-variable reference and the rest through a bulk reference. Check them with `kubectl exec <pod> -- env`.
3. Change a value in the ConfigMap and apply it. **Predict:** does the running Pod see the new value? Verify.
4. Make the Pod pick up the change. Find two ways to do that, one of which works for a Deployment without editing its manifest.

## 2. ConfigMap as files

1. Put the contents of an `index.html` into a ConfigMap. Mount it into an nginx Pod so nginx serves it. Check with `port-forward` and `curl`.
2. Edit the ConfigMap's HTML. **Predict:** does the Pod serve the new page without a restart? How long does it take? Verify, and watch the file with `kubectl exec ... -- cat`.
3. Change the mount to use `subPath` for a single file. Repeat the edit. **Predict** again, then verify. What changed, and why?
4. Mount a ConfigMap over a directory that already has files in the image (for example `/etc/nginx/conf.d`). What happens to the original files?

## 3. Secrets

1. Create a Secret with a username and password. Do it once with `data` and once with `stringData`. What is the difference when you read each back?
2. Run `kubectl get secret <name> -o yaml`, then decode a value with `base64 -d`. Run `kubectl describe secret <name>`. What does each show?
3. Consume the Secret as an environment variable in one Pod and as a mounted file in another. Compare what each one looks like inside the container.
4. In your own words: is base64 encryption? Who can read this Secret, and where is it stored in the cluster? Which component from your Phase 01 notes holds it?

## 4. emptyDir and Pod-scoped storage

1. Write a Pod with two containers that share an `emptyDir`. One writes a timestamp to a file every few seconds. The other reads it.
2. Kill the writing container (see Phase 01 for `crictl`). **Predict:** is the file still there after the container restarts? Verify.
3. Delete the Pod and recreate it. **Predict:** is the file still there? Verify, and explain the difference between the two cases.

## 5. PersistentVolumeClaim

1. Write a PVC with the default StorageClass. Apply it and check `kubectl get pvc`. **Predict:** what is its status right after apply? Verify, and explain what you see using what you learned in exercise 0.
2. Write a Pod that mounts the PVC and writes a file to it. What happens to the PVC status once the Pod is scheduled?
3. Delete the Pod, then recreate it from the same manifest. Is your file still there? Where is the data physically stored? (Look at the PV, and at the node.)
4. Delete the Pod and the PVC. **Predict** what happens to the PV and the data, given the StorageClass's `reclaimPolicy`. Verify.
5. Try to create a PVC that requests a StorageClass that doesn't exist. What status does it get, and where do you find out why?

## 6. StatefulSet

1. Write a headless Service and a StatefulSet with 3 replicas of any image you like, with a `volumeClaimTemplate`. Watch the Pods start with `kubectl get pods -w`. What order do they start in, and what are they named?
2. List the PVCs. What are they named, and how do they relate to the Pods?
3. Write a different file into each Pod's volume. Delete `<name>-1`. **Predict:** what is the new Pod's name, and does it see the same data? Verify.
4. Reach `<name>-0` from another Pod using its stable DNS name. What is the full name format, and which object makes it exist?
5. Scale the StatefulSet to 1, then back to 3. **Predict** what happens to the PVCs and the data of the Pods you removed. Verify.
6. Delete the StatefulSet. What happens to the Pods, and what happens to the PVCs? How would you remove the data too?
7. Compare this with a Deployment of 3 replicas and one shared PVC. What goes wrong, and when would you pick each?

## 7. Debug challenge

Broken manifests are provided in `labs/02-config-and-storage/broken/`. Fix each one using only `kubectl` output, and log the root cause in `notes/02-config-and-storage/debug.md`.

## Done when

- You can write a ConfigMap, a Secret, a PVC, and a StatefulSet from memory.
- You can say which of the ConfigMap consumption methods update in a running Pod, and which don't.
- You can explain what happens to the data when a Pod, a PVC, and a StatefulSet are each deleted.
- Your `notes/02-config-and-storage/` has a note for each object you worked with, and `debug.md` has an entry for every broken case.
