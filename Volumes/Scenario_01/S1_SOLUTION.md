# Volumes - Solutions

## Scenario 1: Clean Creation (emptyDir)

**Diagnosis**
Not applicable. This scenario was for clean creation and muscle memory.

**Root Cause**
Not applicable.

**Fix**
The Pod manifest required an `emptyDir` volume defined under `spec.volumes` and a corresponding `volumeMount` in the container spec pointing to `/usr/share/nginx/html`. 

**Verification**
1. `kubectl get pods` to confirm the `vol-s1-pod` reached the `Running` state.
2. `kubectl exec -it vol-s1-pod -c nginx -- sh` to enter the container. (Note: you initially typed `-c myapp`, which would fail if the container was actually named `nginx`. Always match the exact container name from the manifest).
3. `echo "volume test" > /usr/share/nginx/html/test.txt` and `cat /usr/share/nginx/html/test.txt` to prove the mount is writable and persistent for the life of the Pod.

**Key Takeaways**
- An `emptyDir` volume is created when a Pod is assigned to a node and exists only as long as that Pod is running on that node. If the Pod dies or is rescheduled, the data is erased.
- Multiple containers in the same Pod can mount the same `emptyDir` to share files.
- Always verify the exact container name in the manifest when using `kubectl exec -c <name>`.

***

