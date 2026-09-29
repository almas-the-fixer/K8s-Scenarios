## Deployment — Scenario 1 (Easy)

**Goal:** create a Deployment, then use it for the thing a bare ReplicaSet can't do — actually roll out a change.

**Suggested time:** 10 minutes

### Task

1. Create a Deployment named `shop-web` with **3 replicas**, running image `nginx:1.24`. Pods must carry the label `app=shop-web`.
2. Confirm all 3 Pods are `Running`, and find the name of the ReplicaSet the Deployment created for you.
3. Update the image to `nginx:1.25` using an imperative command (not by editing the file), and watch the rollout happen. Don't just check the end state — watch it transition.
4. Once it's done, confirm two things:
   - the new ReplicaSet from step 3 has all 3 Pods
   - the ReplicaSet from step 2 still exists, but now has **0** Pods
5. Check the rollout history, then roll back to the previous version (`nginx:1.24`) using a `kubectl` command, not by re-editing the image yourself.
6. Confirm the rollback worked by checking the image nginx is actually running now, not just that the command succeeded.

<details>
<summary>Hint (step 3)</summary>

`kubectl set image` targets a Deployment's container by name, not the Deployment's own name.

</details>

<details>
<summary>Hint (step 5)</summary>

`kubectl rollout` has subcommands for both viewing history and undoing it.

</details>

### Deliverable

Deployment manifest as `s1_deployment.yml`. No broken state in this one — it's to get rollout, rollback, and the Deployment→ReplicaSet relationship into muscle memory before we start breaking things in S2.