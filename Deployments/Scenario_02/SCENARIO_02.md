## Deployment — Scenario 2 (Troubleshooting)

**Goal:** you have a healthy Deployment. You push an update, and the rollout stalls. Work out why, decide the right response under pressure, and verify you actually fixed it.

**Suggested time:** 15 minutes

### Setup

Apply the files yourself, **in this order**:

1. `s2_deployment.yml` — a healthy baseline Deployment. Confirm it's fully up before moving on.
2. `s2_deployment_bad.yml` (marked with a warning comment at the top) — the "update" you're pushing.

Don't edit the bad file. Copy it to `s2_deployment_fixed.yml` and fix things there.

### Task

1. After applying the bad file, run:
```bash
   kubectl rollout status deployment/shop-api --timeout=15s
```
It won't finish in time. Note what that tells you, and don't just wait longer.

2. Run `kubectl get rs` and `kubectl get pods`. Explain, in your own words, why **both** the old and new ReplicaSets have Pods right now, and why the Deployment isn't just stuck at "creating."
3. Diagnose the actual failure with `describe`, same as before.
4. **Decision point:** the rollout is stuck and you're not 100% sure yet what's wrong. What's the safer first move — keep digging, or get back to a known-good state immediately? Do that first.
5. Once you're back on a known-good state, confirm it with `kubectl rollout status` (should complete cleanly this time) and check which image is actually running.
6. Now fix the real bug in `s2_deployment_fixed.yml`, apply it, and prove the fix works — that means a rollout that reaches `kubectl rollout status` success, not just Pods that look okay at a glance.

<details>
<summary>Hint (step 2)</summary>

Think about what `maxUnavailable` guarantees. The Deployment won't kill old Pods faster than new ones become ready — so if new ones never become ready, what happens to the old ones?

</details>

<details>
<summary>Hint (step 4)</summary>

You already know a command from Scenario 1 that gets you back to the last working revision without needing to know the root cause first.

</details>