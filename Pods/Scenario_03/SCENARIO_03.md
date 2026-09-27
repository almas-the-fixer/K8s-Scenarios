## Pod — Scenario 3 (Hard — wrap-up)

**Goal:** a multi-container Pod with **three separate, stacked bugs**. Fixing one will reveal the next — don't assume you're done after the first fix. This is intentionally harder than a typical CKA Pod question.

**Suggested time:** 15 minutes

### Setup

A script will run `kubectl apply -f s3_pod.yml` for you. Expect the Pod to go through multiple different bad states before it's actually healthy — recognizing *which* failure mode you're looking at (Pending vs CrashLoopBackOff vs Running-but-restarting) is part of the exercise.

### Task

1. Apply it and observe the first failure mode. Diagnose it — don't guess, use `kubectl describe`/`kubectl get events --sort-by=.lastTimestamp`.
2. Fix the minimum needed, re-apply/re-check, and see if it's actually healthy or if a *different* problem shows up.
3. Repeat until the Pod is genuinely `Running` **and** `Ready`, and stays that way (watch it for a bit — one of the bugs won't show up immediately).
4. Once stable, confirm the main container can actually read the shared file it depends on.

<details>
<summary>### Hints (only if stuck)</summary>

- One bug is about where the Pod is even allowed to be scheduled.
- One bug involves two containers **not actually sharing** what you'd assume they share.
- One bug won't show up until *after* the container looks like it's running fine.

</details>