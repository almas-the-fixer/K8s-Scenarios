## ConfigMap — Scenario 2 (Troubleshooting)

**Goal:** a Pod that's `Running`, no restarts, no error events — and still completely wrong. This is a different kind of bug than you've debugged so far: nothing is loudly failing.

**Suggested time:** 10 minutes

### Setup

Apply the files yourself, in order:

1. `s2_configmap.yml`
2. `s2_pod.yml` (marked with a warning comment at the top)

Don't edit the buggy file. Copy it to `s2_pod_fixed.yml` and fix things there.

### Task

1. Check the Pod's status. It'll look completely healthy — confirm that for yourself.
2. Check the logs. Something's off, even though nothing crashed and no events fired.
3. Before touching the Pod spec, run `kubectl describe configmap app-settings` and compare its actual keys against what the Pod's manifest references, side by side.
4. Fix the mismatch in `s2_pod_fixed.yml`, re-apply, and confirm the logs now show the correct value.
5. In your own words: why did this fail silently instead of producing a `CreateContainerConfigError` like the ConfigMap bugs you've hit before? There's one specific field responsible.
6. When would that field actually be the right choice to use, versus when is it dangerous?

<details>
<summary>Hint (step 3)</summary>

Don't guess the typo by eye. Pull the real keys from the live object and diff them against the manifest.

</details>