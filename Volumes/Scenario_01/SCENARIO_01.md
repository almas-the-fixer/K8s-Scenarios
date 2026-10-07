# Scenario 1: Volumes (emptyDir)

## Objective
Clean creation and verification of a Pod with an attached emptyDir volume. No bugs. Purpose: get the core volume definition and mount syntax into muscle memory.

## Tasks
1. Write a Pod manifest for a Pod named `vol-s1-pod`.
2. The Pod should run a single container named `nginx` using the `nginx:alpine` image.
3. Define an `emptyDir` volume named `data-vol`.
4. Mount `data-vol` inside the container at `/usr/share/nginx/html`.
5. Apply your YAML file.
6. Verify the Pod reaches the `Running` state.
7. Exec into the running container and create a file named `test.txt` inside the mounted directory (`/usr/share/nginx/html`) with the content "volume test".
8. Verify the file exists and contains the correct text.

<details>
<summary>Hint</summary>

To verify the mount from the outside without exec-ing, you can check the Pod's description. Look for the "Mounts" section in the container details.
</details>