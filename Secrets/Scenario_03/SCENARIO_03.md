## Secret — Scenario 3 (Hard — wrap-up)

**Goal:** two Secrets feeding one Pod, one of them immutable, plus a permissions detail on the mounted files that's easy to overlook.

**Suggested time:** 15 minutes

### Setup

Apply these yourself, in order:

1. `s3_secret_primary.yml` (immutable)
2. `s3_secret_override.yml`
3. `s3_pod_bad.yml`

### Task

1. Confirm the Pod/Deployment starts cleanly. The selector and Pod template labels match, so this is just a normal create — nothing to diagnose here.
2. Both Secrets define `API_KEY` via `envFrom`. Predict which value wins **before** checking logs, same as you did in ConfigMap S3 — then confirm. State the rule, not just the answer.
3. Once healthy, check the file permissions on the mounted Secret files. `ls -l` on the mount path itself shows the **symlink's** permissions, which are always `777`-equivalent and tell you nothing real — follow it to the actual target:
```bash
   kubectl exec <pod> -- ls -la /etc/secret/..data/
```
   Compare that against a ConfigMap-mounted file from an earlier scenario. They default to the **same** mode. Explain why that might be more surprising than expected, and what the real security boundary for Secrets actually is, if it isn't file permissions.
4. **`API_KEY` needs to rotate.** `db-primary` (the Secret actually feeding the winning value) is immutable, so you can't edit it in place. Rotate it correctly:
   - Edit the value in `s3_secret_primary.yml` itself.
   - Delete the existing `db-primary` Secret.
   - Re-apply the edited file to recreate it under the same name.
   - Note: `db-override` and `db-primary` are two **separate, independently-named objects** — they only happen to share a key name (`API_KEY`). Changing one never touches the other. Make sure you're rotating the one that actually matters here.
5. Roll the Deployment so a fresh Pod picks up the rotated Secret, then confirm **both** the mounted file and the env var reflect the new value. Tie this back to what you already know:
   - the mounted file updates because this scenario doesn't use `subPath` (unlike ConfigMap S3), so the normal symlink-swap sync still applies
   - the env var only updates because the Pod itself was recreated — not because env vars ever refresh on their own (same rule since ConfigMap S1 and Secret S2)

<details>
<summary>Hint (step 3)</summary>

A symlink's own listed permissions are meaningless — always follow it. The real control for who can read a Secret's contents happens before anything is ever mounted: it's about API-level access to the Secret object itself.

</details>

<details>
<summary>Hint (step 4)</summary>

If deleting one Secret and reapplying a file doesn't fix the value, check carefully which Secret object that file actually defines. Two Secrets can share a key name while being entirely unrelated objects.

</details>