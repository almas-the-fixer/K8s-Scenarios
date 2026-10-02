## ConfigMap — Scenario 3 (Hard — wrap-up)

**Goal:** two ConfigMaps feeding one Pod multiple ways — bulk env vars with a value collision, and a single file mounted via `subPath`. One failure blocks the Pod outright; the other two are about understanding *rules*, not just fixing typos.

**Suggested time:** 20 minutes

### Setup

Apply these yourself, in order:

1. `s3_configmap_primary.yml`
2. `s3_configmap_override.yml`
3. `s3_pod_bad.yml` (marked with a warning comment at the top)

Don't edit the buggy file. Copy it to `s3_pod_fixed.yml` and fix things there.

### Task

1. The Pod won't come up healthy. Diagnose it the usual way — `describe`/events — and fix just that one problem in `s3_pod_fixed.yml`. Re-apply.
2. **Before checking anything else**, look at the Pod spec's `envFrom` list — it references **both** ConfigMaps, and both define `DB_HOST` with different values. Without running anything yet, predict which value will win, and write down *why* you think so.
3. Now check the logs and see which value actually won. Were you right? State the actual rule governing what happens when `envFrom` pulls from multiple sources that define the same key.
4. Confirm the mounted file is correct too — `cat` it and compare against the ConfigMap's content.
5. **Live-update test:** patch the ConfigMap key that feeds the mounted file (not the whole ConfigMap — just that one key) to a new value. Wait a minute, then check the mounted file again. Does it update? Compare this against what you saw in ConfigMap S1 with a plain (non-`subPath`) volume mount, and explain the difference.
6. While you're at it, patch `DB_HOST` too and confirm the env var still doesn't move — same frozen-at-start behavior as S1, just double-checking it holds here too.

<details>
<summary>Hint (step 1)</summary>

Same category of ConfigMap-volume bug as something you've already diagnosed — just check `items` carefully against the ConfigMap's real keys, instead of a plain `env` key.

</details>

<details>
<summary>Hint (step 5)</summary>

`subPath` doesn't just change *where* the file lands — it changes *how* Kubernetes keeps it in sync. Look up what `subPath` actually does differently from a plain ConfigMap volume mount.

</details>