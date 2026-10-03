## Secret — Scenario 2 (Troubleshooting)

**Goal:** a Secret marked `immutable: true`, and an "update" that gets flatly rejected. This mirrors the Deployment-selector and Pod-field immutability bugs you've already hit, but it's a feature specific to Secrets (and ConfigMaps) worth knowing on its own.

**Suggested time:** 8 minutes

### Setup

Apply the files yourself, in order:

1. `s2_secret.yml` — baseline, immutable, with `DB_PASS=initial-pass`.
2. `s2_pod.yml` — consumes it as an env var. Confirm healthy.
3. Attempt `s2_secret_update.yml` (marked with a warning comment at the top) — a routine-looking password rotation.

Don't edit the buggy file. Copy it to `s2_secret_update_fixed.yml` if you end up needing a different approach entirely (you might not — read the error carefully first).

### Task

1. Apply the update file. It'll be rejected. Read the actual error — which field is `immutable` actually locking, and why would anyone deliberately want that lock in production, given that Secrets often need to rotate?
2. You can't patch your way out of this. Work out the correct way to actually rotate this value without ever disabling `immutable`, and do it.
3. Confirm the Pod picks up the new value — but think carefully about *how* it would need to, given everything you learned in ConfigMap S1 about when env vars actually refresh.
4. Prove the old Secret object is genuinely gone, not just superseded.

<details>
<summary>Hint (step 2)</summary>

If a field can't be changed on an existing object, what's the actual way to get new content live? You've used this approach before, more than once.

</details>