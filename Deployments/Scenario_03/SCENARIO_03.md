## Deployment — Scenario 3 (Hard — wrap-up)

**Goal:** three separate, stacked failures, each a genuinely different *category* from anything you've hit so far in Deployment or ReplicaSet. Fixing one reveals the next.

**Suggested time:** 20 minutes

### Setup

Apply these yourself, in order:

1. `s3_secret.yml`
2. `s3_deployment.yml` — a healthy baseline. Confirm 3/3 before moving on.
3. Attempt `s3_deployment_bad.yml` (marked with a warning comment at the top) as if it were your next update.

Don't edit the bad file. Copy it to `s3_deployment_fixed.yml` and fix things there.

### Task

1. Applying the bad file will be **rejected outright** — you won't even get to see a Pod. Read the actual error message. Which field is it complaining about, and why would Kubernetes refuse to let you change that field after a Deployment already exists?
2. Fix only what's needed to get the apply *accepted* (not necessarily healthy yet) in `s3_deployment_fixed.yml`. Re-apply and confirm it goes through this time.
3. A new failure shows up at the container level. Diagnose it with `describe`/events, fix it in the same file, re-apply.
4. A **third**, different failure appears. Its `Reason` in `describe` won't look like anything from S1 or S2. Diagnose it, and explain *why* the value that caused it is wrong — this one isn't a typo, it's a number that's just too small for what the container needs to do.
5. Fix it, reach 3/3 Ready, and confirm with `kubectl rollout status` that this rollout completed cleanly.
6. Final check: confirm the Secret's value is actually visible inside a running container — without dumping the whole environment.

<details>
<summary>Hint (step 1)</summary>

Not every field in a Deployment's spec can be changed after creation. Some are permanent by design once the object exists. The error message names the exact field.

</details>

<details>
<summary>Hint (step 4)</summary>

This isn't a wrong path and it isn't a wrong key. Look at the actual numbers you gave the container to work with, and think about what nginx needs just to start.

</details>