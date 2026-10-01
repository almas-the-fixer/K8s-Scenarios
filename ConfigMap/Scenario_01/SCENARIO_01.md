## ConfigMap — Scenario 1 (Easy)

**Goal:** get a ConfigMap into a Pod three different ways in one exercise — as individual env vars, as a bulk env source, and as mounted files — so you can feel the difference between them before S2 breaks one.

**Suggested time:** 10 minutes

### Task

1. Create a ConfigMap named `app-config` with these keys:
   - `LOG_LEVEL=debug`
   - `MAX_CONNECTIONS=100`
   - `app.properties` with multi-line content of your choice (at least 2 lines, e.g. `timeout=30\nretries=3`)
2. Create a Pod named `config-demo` (image `busybox:1.36`, command that just sleeps so you can exec in) that consumes this ConfigMap **three different ways at once**:
   - `LOG_LEVEL` injected as a single named env var, referencing just that one key
   - **all** of `LOG_LEVEL` and `MAX_CONNECTIONS` injected automatically as env vars, without naming each one individually
   - the whole ConfigMap mounted as a volume at `/etc/config`
3. Once running, confirm all three mechanisms actually worked:
   - print the single named env var
   - print the bulk-injected env vars (both of them, without printing the entire environment)
   - `cat` the mounted `app.properties` file and confirm it matches what you put in the ConfigMap
4. Without restarting the Pod: edit the ConfigMap's `LOG_LEVEL` value using `kubectl edit`, then check whether the **mounted file** version and the **env var** version reflect the change. Report what you find for each — they won't necessarily behave the same way.

### Deliverable

ConfigMap as `s1_configmap.yml`, Pod as `s1_pod.yml`.

<details>
<summary>Hint (step 2, bulk env vars)</summary>

There's a field for referencing an entire ConfigMap as a source of env vars, separate from the field you'd use to reference one key at a time.

</details>

<details>
<summary>Hint (step 4)</summary>

Volume-mounted ConfigMaps and env-var ConfigMaps are populated through two different mechanisms under the hood, and only one of them checks back periodically.

</details>