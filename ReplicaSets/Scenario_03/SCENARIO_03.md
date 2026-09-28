## ReplicaSet — Scenario 3 (Hard — wrap-up)

**Goal:** a ReplicaSet with **stacked problems**. Fixing one exposes the next, and the fixes only take effect if you handle how a ReplicaSet treats existing Pods. Then you'll take one Pod out of the ReplicaSet's control on purpose.

**Suggested time:** 20 minutes

### Setup

Apply the files yourself, **in this order**:

1. `s3_configmap.yml`
2. `s3_replicaset.yml` (the buggy ReplicaSet, marked with a warning comment at the top)

Don't edit the buggy file. Copy it to `s3_replicaset_fixed.yml` and fix things there. Don't edit the ConfigMap either.

### Task

1. After applying both, run `kubectl get pods`. Diagnose the first failure using `describe` or events, and note the exact error message.
2. Fix it in `s3_replicaset_fixed.yml` and apply it **over the running ReplicaSet** (don't delete the ReplicaSet). Get past this failure.
3. A **different** problem will show up. Diagnose it. Notice how its symptoms differ from the first one, in the `STATUS` and `READY` columns.
4. Fix it and reach **4/4 Ready**. Whenever you need to replace Pods, do it with a **single command** that covers all of them. Don't delete them one by one by name.
5. **Quarantine:** pick one healthy Pod and take it out of the ReplicaSet's control **without deleting it and without editing the ReplicaSet**. Show that:
   - the ReplicaSet immediately created a replacement (`get rs` still shows 4 desired / 4 current / 4 ready, while `get pods` shows 5 Pods)
   - the quarantined Pod is still `Running` and has no owner
6. **Predict, then test:** if you put the quarantined Pod's label back to its original value, what happens? Write your prediction *before* you try it. Then do it and see whether you were right.

<details>
<summary>Hint (step 2)</summary>

Think back to Scenario 2. Does the ReplicaSet re-read its template for Pods that already exist?

</details>

<details>
<summary>Hint (step 4)</summary>

`kubectl delete` can select Pods by label instead of by name.

</details>

<details>
<summary>Hint (step 5)</summary>

A ReplicaSet finds its Pods by label. What happens to a Pod that stops matching?

</details>