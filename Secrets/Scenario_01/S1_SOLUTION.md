# Secret — Solutions

## Scenario 1

### Create and verify

```bash
kubectl apply -f s1_secret.yml
kubectl apply -f s1_pod.yml
```

**Env var:**
```bash
kubectl exec secret-demo -- printenv DB_PASS
```
```
s3cr3t
```

**Mounted file:**
```bash
kubectl exec secret-demo -- cat /home/db/DB_USER
kubectl exec secret-demo -- cat /home/db/DB_PASS
```
```
admin
s3cr3t
```

### Base64 is encoding, not encryption

```bash
kubectl get secret db-creds -o yaml
```
```yaml
data:
  DB_PASS: czNjcjN0
  DB_USER: YWRtaW4=
```
```bash
echo "czNjcjN0" | base64 -d
```
```
s3cr3t
```
Anyone with read access to the Secret object can decode it with a single, standard command — no key, no password, nothing secret about the encoding itself. Base64 exists so arbitrary binary-safe data can be stored as text in YAML/JSON, not to hide the value. Real protection comes from RBAC controlling who can `get`/`describe` the Secret object in the first place (covered later in the roadmap), not from the encoding.

### A small syntax trap along the way

`secretKeyRef` (like `configMapKeyRef`) is a single object, not a list — it never takes a leading `-`. The way to confirm this without guessing: `kubectl explain <path>` prints the field's type on its `FIELD:` line. `<Object>` (no brackets) means no dash; `<[]Object>` (with brackets) means it's a list and each entry needs one. `containers` is a real example of a true list (`<[]Object>`), which is why each one starts with `-` — `secretKeyRef` will always show `<Object>`.

### Key takeaways

- Secret and ConfigMap behave identically for env/volume consumption — the only real difference is that Secret values are base64-encoded at rest, which is **encoding for safe storage, not encryption**. Treat Secret contents as just as sensitive as ConfigMap data would be if it contained passwords.
- `kubectl explain <field.path>`'s `FIELD:` line type annotation (`<Object>` vs `<[]Object>`) is the fast, reliable way to check whether a field needs a `-` in front of it, instead of guessing from YAML examples.