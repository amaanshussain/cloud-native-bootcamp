---
tags:
  - Lab
  - Intermediate
  - Storage
  - OpenShift
  - Kubernetes
---

# Persistent Storage Lab

<span class="lab-badge">30 min</span> <span class="lab-badge">Intermediate</span>

In this lab you'll dynamically provision persistent storage using a PersistentVolumeClaim (PVC), attach it to a PostgreSQL database Pod, insert data, and prove that the data survives Pod deletion and restart.

This lab uses a **tutorial reference** format. Each step gives you a goal and points to the section of the [PersistentVolumes and Claims](../../../openshift/state-persistence/pv-pvc.md), [State Persistence Overview](../../../openshift/state-persistence/index.md), [Container Configuration](../../../openshift/configuration/index.md), or [Projects/Namespaces](../../../openshift/core-concepts/namespaces-projects/index.md) documentation that shows the command or syntax you need. **You build the manifests and commands yourself** by adapting the documentation examples to this lab. When the documentation doesn't cover something, the step gives you the command and explains what it does.

!!! tip "Keep the documentation open"
    Open [PersistentVolumes and Claims](../../../openshift/state-persistence/pv-pvc.md) in a second tab before you start. Omitting `storageClassName` causes the cluster's default StorageClass to dynamically provision the volume for your claim.

## Prerequisites

- You're logged in to an OpenShift cluster with `oc` (or an upstream Kubernetes cluster with `kubectl`). See [Lab Environments](../../../lab-environments.md).
- A terminal and a text editor.

## Part 1: Create a project and check storage classes

1. Create a new project named `lab5` and switch to it.
    - **Reference:** [Projects/Namespaces, References](../../../openshift/core-concepts/namespaces-projects/index.md#references). Create `lab5` instead of `dev`.
    - **Check:** `oc project` shows `Using project "lab5"`.

2. Verify that a default StorageClass is available in the cluster.
    - **Reference:** [PersistentVolumes and Claims, Dynamic provisioning](../../../openshift/state-persistence/pv-pvc.md#dynamic-provisioning-the-usual-way) explains dynamic provisioning via StorageClasses. The documentation doesn't cover listing storage classes, so use this command:

    ```bash
    oc get storageclass
    ```

    - **Check:** one StorageClass is marked `(default)`.

## Part 2: Create a PersistentVolumeClaim

3. Write a PersistentVolumeClaim manifest named `postgresql-pvc.yaml` requesting `1Gi` of storage with `ReadWriteOnce` access mode.
    - **Reference:** [PersistentVolumes and Claims, Dynamic provisioning](../../../openshift/state-persistence/pv-pvc.md#dynamic-provisioning-the-usual-way) shows the `PersistentVolumeClaim` YAML structure. Request `storage: 1Gi` and access mode `ReadWriteOnce`.
    - **Check:** your `postgresql-pvc.yaml` defines `kind: PersistentVolumeClaim`, `name: postgresql-pvc`, `accessModes: ["ReadWriteOnce"]`, and `resources.requests.storage: 1Gi`.

4. Apply `postgresql-pvc.yaml` to create the claim.
    - **Reference:** [PersistentVolumes and Claims, References](../../../openshift/state-persistence/pv-pvc.md#references) shows how to view claims with `oc get pvc`.
    - **Check:** `oc get pvc postgresql-pvc` lists the PVC (it may show `Pending` or `Bound` depending on whether the StorageClass uses `WaitForFirstConsumer`).

## Part 3: Deploy PostgreSQL with persistent storage

5. Create a Pod manifest named `postgresql-pod.yaml` running `quay.io/sclorg/postgresql-16-c9s` that mounts `postgresql-pvc` at `/var/lib/pgsql/data`.
    - **Reference:** [PersistentVolumes and Claims, Dynamic provisioning](../../../openshift/state-persistence/pv-pvc.md#dynamic-provisioning-the-usual-way) shows mounting a PVC into a Pod. [Container Configuration, Environment Variable](../../../openshift/configuration/index.md#environment-variable) shows configuring environment variables.
    - Configure the Pod with:
        - `metadata.name: postgresql-pod` and `metadata.labels.app: postgresql`
        - Container name `postgresql`, image `quay.io/sclorg/postgresql-16-c9s`, `containerPort: 5432`
        - Environment variables `POSTGRESQL_USER: rebel`, `POSTGRESQL_PASSWORD: password`, and `POSTGRESQL_DATABASE: deathstar`
        - Volume referencing claim `postgresql-pvc` mounted at `/var/lib/pgsql/data`
    - **Check:** your manifest mounts the volume at `/var/lib/pgsql/data` and configures all three PostgreSQL environment variables.

6. Apply `postgresql-pod.yaml` and verify that the Pod is `Running` and the PVC is `Bound`.
    - **Reference:** [Pod Basics, References](../../../openshift/pods/index.md#references) shows applying manifests and listing pods.
    - **Check:** `oc get pvc,pod` shows `persistentvolumeclaim/postgresql-pvc` with status `Bound` and `pod/postgresql-pod` with status `Running`.

## Part 4: Test data persistence across Pod restarts

7. Write a record to the PostgreSQL database inside the running Pod.
    - **Reference:** [Pod Basics, References](../../../openshift/pods/index.md#references). Run `psql` using `oc exec`:

    ```bash
    oc exec postgresql-pod -- psql -d deathstar -c "CREATE TABLE plans (weakness text); INSERT INTO plans VALUES ('thermal exhaust port');"
    ```

    - **Check:** output shows `CREATE TABLE` and `INSERT 0 1`.

8. Delete `postgresql-pod`, recreate it, and verify that the database table and data persisted.
    - **Reference:** [Pod Basics, References](../../../openshift/pods/index.md#references) shows `oc delete pod` and `oc apply -f`. Recreate the Pod and query the table:

    ```bash
    oc delete pod postgresql-pod
    oc apply -f postgresql-pod.yaml
    oc wait --for=condition=Ready pod/postgresql-pod --timeout=120s
    oc exec postgresql-pod -- psql -d deathstar -c "SELECT * FROM plans;"
    ```

    ```text title="Expected output"
          weakness
    ----------------------
     thermal exhaust port
    (1 row)
    ```

    - **Check:** `SELECT * FROM plans;` returns `thermal exhaust port`, proving the data survived Pod deletion.

## Cleanup

Delete the `lab5` project and all resources created in this lab.

- **Reference:** [Projects/Namespaces, References](../../../openshift/core-concepts/namespaces-projects/index.md#references) shows project management. To delete the project:

```bash
oc delete project lab5
```

| Part | What it does |
| --- | --- |
| `oc delete project lab5` | Deletes the `lab5` project, PVC, and Pod |

## What you learned

- [x] Requesting storage dynamically with a PersistentVolumeClaim
- [x] Attaching a PersistentVolumeClaim to a container volume mount
- [x] Verifying data survival across container and Pod deletion cycles
- [x] Configuring database credentials via container environment variables
