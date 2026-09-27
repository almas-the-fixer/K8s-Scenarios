## Pod — Scenario 2 (Troubleshooting)

**Goal:** given a Pod that's already applied and misbehaving, diagnose and fix it — no rewriting from scratch, find the actual bug.

**Suggested time:** 8 minutes

### Setup

A script will run `kubectl apply -f s2_pod.yml` for you. The Pod will not reach `Running`. Your job: find out why using kubectl's diagnostic commands, fix the manifest, and re-apply until it's healthy.

### Task

1. Apply it (or let the script apply it) and observe that it doesn't come up healthy.
2. Diagnose **why**, using `kubectl describe` / `kubectl get events` — don't just guess.
3. Fix it with the **minimum change needed** to get it Running — you're allowed to create whatever supporting object is missing, but don't change the Pod's fundamental design (still a busybox reading a mounted config file).
4. Once healthy, confirm the container actually printed the config file contents in its logs.