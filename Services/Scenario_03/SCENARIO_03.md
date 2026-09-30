## Service — Scenario 3 (Hard — wrap-up)

**Goal:** a Service with **two ports** — one completely healthy, one broken in three different ways. Use the healthy port as your reference point for spotting what's different about the broken one.

**Suggested time:** 20 minutes

### Setup

Apply these yourself, in order:

1. `s3_deployment.yml` — a healthy baseline with a container exposing two **named** ports.
2. `s3_service_bad.yml` (marked with a warning comment at the top) — the broken Service.

Don't edit the broken file. Copy it to `s3_service_fixed.yml` and fix things there.

### Task

1. Applying the bad file will be **rejected outright**, same category as Deployment S3's immutable-field bug — but a different rule entirely. Read the actual error. What range is being enforced, and why would Kubernetes restrict this particular field to a narrow range instead of letting you pick anything?
2. Fix only what's needed to get the apply accepted, in `s3_service_fixed.yml`. Confirm it goes through.
3. Try reaching the Service on the `web` port. It'll fail. Before touching anything, run `kubectl get endpoints` and compare the `web` port's entry against the `metrics` port's entry side by side. What's different about them?
4. Fix that difference. Re-apply, re-check endpoints — `web` should now look like `metrics` did all along. Try reaching it again. It **still** fails, but differently. Describe exactly how the failure is different this time (refused vs something else).
5. Diagnose that third failure by comparing the two port blocks in your YAML directly against each other, field by field. Fix it.
6. Reach `web` successfully, by both ClusterIP and NodePort. Then confirm `metrics` — the port you never touched — still works too, proving your fixes to `web` didn't collide with it.

<details>
<summary>Hint (step 3)</summary>

`targetPort` can reference a port by number or by the `name` field on the container's port. Check whether the name on each side actually matches, on both ports.

</details>

<details>
<summary>Hint (step 4)</summary>

"Connection refused" means something answered and said no. A very different kind of silence means nothing answered at all. Which one is this, and what single field controls which protocol kube-proxy sets up rules for?

</details>