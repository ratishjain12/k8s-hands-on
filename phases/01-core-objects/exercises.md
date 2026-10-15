# Phase 01 exercises

Write every manifest by hand in `labs/01-core-objects/<exercise>/`. For each exercise, write your prediction down before running anything.

## 0. Explore the cluster

1. Run `kubectl get pods -A -o wide`. List every pod in `kube-system` and write one sentence on what you think it does.
2. Run `kubectl get nodes -o wide` and `kubectl describe node <name>`. Which fields tell you what the node can run?
3. Run `docker ps`. How does this relate to the node you just described?

## 1. A bare Pod

1. Write a Pod manifest running `nginx`. Apply it.
2. Predict: what is its status right after apply? Verify.
3. Delete the Pod. Predict: does it come back? Verify, then explain why.
4. Reach it from your machine without a Service.

## 2. Deployment and self-healing

1. Write a Deployment with 3 replicas of nginx.
2. Delete one Pod while watching with `kubectl get pods -w`. Predict what happens, then verify.
3. Scale to 5, then to 0, using two different methods (imperative and by editing the manifest).
4. Find the ReplicaSet the Deployment created. What is its name made of?

## 3. Rolling update and rollback

1. Change the image tag to a specific older version. Apply and watch the rollout.
2. Apply an image tag that does not exist. Predict what happens to the running Pods, then verify.
3. Diagnose it using only `describe` and `events`.
4. Roll back with `kubectl rollout`.

## 4. Services

1. Expose the Deployment inside the cluster with a Service. Prove it works from another Pod.
2. Predict: what happens to the Service when you delete a backing Pod? When you scale to 0?
3. Change a Pod label so the Service stops selecting it. Observe `kubectl get endpoints`.
4. Reach the Service from your machine two different ways.

## 5. Namespaces

1. Create a namespace and deploy the same app in it with the same names.
2. Reach a Service in another namespace by DNS name. What is the full name format?

## 6. Debug challenge

Your instructor (or a contributor) provides broken manifests in `labs/01-core-objects/broken/`. Fix each one using only `kubectl` output, and log the root cause in `notes/01-core-objects/debug.md`.

## Done when

- You can write a Deployment and a Service from memory.
- You can explain, without notes, why deleting a Pod managed by a Deployment doesn't delete the app.
- Your `notes/01-core-objects/` has a note for each object you worked with, and `debug.md` has at least five entries.
