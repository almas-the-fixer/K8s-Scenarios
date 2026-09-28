## ReplicaSet — Scenario 1 (Easy)

**Goal:** create a ReplicaSet, watch it keep its Pod count on target, and practice scaling it two different ways.

**Suggested time:** 8 minutes

### Task

1. Create a ReplicaSet named `web-rs` with **3 replicas**, running image `nginx:1.25`. The Pods it manages must carry the label `app=web`, and the ReplicaSet's selector must match that label.
2. Confirm all 3 Pods are `Running` and show which node each one landed on.
3. Delete **one** of the Pods by name. Show that the ReplicaSet replaced it, and how you can tell the replacement is a *new* Pod rather than the old one recovering.
4. Scale the ReplicaSet to **5** replicas using a `kubectl` command (without editing the file), and confirm 5 Pods are running.
5. Now scale it down to **2** by changing the manifest and re-applying it. Confirm only 2 Pods remain.
6. Print only the ReplicaSet's desired, current, and ready replica counts (not the full `describe` output).

### Deliverable

ReplicaSet manifest as `s1_replicaset.yml`. No broken state in this one. It's just to get the create, verify, and scale commands into muscle memory.

<details>
<summary>Hint (step 6)</summary>

`kubectl get rs` already shows these three numbers in its default output.

</details>