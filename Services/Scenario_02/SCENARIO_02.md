## Service — Scenario 2 (Troubleshooting)

**Goal:** a Service that exists, has endpoints, and still doesn't work. Work out why "endpoints exist" isn't the same as "it's configured right."

**Suggested time:** 10 minutes

### Setup

Apply the files yourself, in order:

1. `s2_deployment.yml` — a healthy baseline.
2. `s2_service_bad.yml` (marked with a warning comment at the top) — the broken Service.

Don't edit the broken file. Copy it to `s2_service_fixed.yml` and fix things there.

### Task

1. Try reaching the Service from a debug Pod, the same way you did in S1. It won't work — note exactly what happens (refused? times out? something else?).
2. Before assuming the selector is wrong, **check the endpoints first**. What do you find, and what does that rule out?
3. Now check what port the container is actually listening on — don't guess, confirm it. Compare that against what the Service is configured to send traffic to.
4. Fix it in `s2_service_fixed.yml`, apply, and confirm the same debug-Pod check now succeeds.
5. In your own words: what's the actual difference between a Service having *no endpoints* and a Service having endpoints that are *pointed at the wrong port*? Which one would `kubectl get endpoints` catch, and which one wouldn't be obvious from that command alone?

<details>
<summary>Hint (step 3)</summary>

`kubectl describe pod <name>` shows the container's declared port. You can also exec in and check what the process is actually bound to.

</details>