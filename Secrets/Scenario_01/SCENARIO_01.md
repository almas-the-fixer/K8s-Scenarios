## Secret — Scenario 1 (Easy)

**Goal:** create a Secret, consume it as env vars and a mounted file, and verify the base64 encoding isn't encryption.

**Suggested time:** 8 minutes

### Task

1. Create a Secret named `db-creds` (type `Opaque`) with keys `DB_USER=admin` and `DB_PASS=s3cr3t`, using `stringData` (not pre-encoded `data`).
2. Create a Pod `secret-demo` (busybox, sleep) consuming it two ways: `DB_PASS` as a named env var, and the whole Secret mounted at `/etc/secret`.
3. Confirm both: print the env var, and `cat` the mounted file for `DB_USER`.
4. Run `kubectl get secret db-creds -o yaml` and decode `DB_PASS` from the raw output yourself (base64), without using `-o jsonpath` to extract it for you. Confirm it matches what you set.

### Deliverable

`s1_secret.yml`, `s1_pod.yml`.