## ReplicaSet — Scenario 2 (Troubleshooting)

**Goal:** a ReplicaSet that comes up unhealthy, plus one Pod you didn't expect. Work out what's going on, and find out what a ReplicaSet does and doesn't do when you fix it.

**Suggested time:** 12 minutes

### Setup

Apply the files **in this order**:

1. `s2_stray_pod.yml` (a standalone Pod)
2. `s2_replicaset.yml` (the buggy ReplicaSet, marked with a warning comment at the top)

Don't edit the buggy file. Copy it to `s2_replicaset_fixed.yml` and fix things there.

### Task

1. After applying both, run `kubectl get pods`. The ReplicaSet asks for 3 replicas. Work out how many Pods the ReplicaSet **actually created itself**, and explain the total you see.
2. Prove your explanation with a command. Don't just infer it from the names.
3. Some Pods are unhealthy. Find out **why** using `describe` and events, not guesses.
4. Fix the manifest in `s2_replicaset_fixed.yml` and apply it. Then check whether the ReplicaSet is actually healthy. If it isn't, work out why and get to **3 Ready Pods** without deleting the ReplicaSet.
5. Finish with a command that shows the standalone Pod is now managed by the ReplicaSet.

<details>
<summary>Hint (step 2)</summary>

Every Pod records who owns it in its metadata. Look at the full object, not the summary table.

</details>

<details>
<summary>Hint (step 4)</summary>

Think about when a ReplicaSet creates Pods. Does it re-check the template on Pods that already exist?

</details>