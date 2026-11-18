---
tags:
  - Lab
  - Intermediate
  - Configuration
  - Security
  - OpenShift
  - Kubernetes
---

# Pod Configuration Lab

<span class="lab-badge">30 min</span> <span class="lab-badge">Intermediate</span>

In this lab you'll assemble comprehensive Pod configurations in OpenShift: mounting external configuration files from a ConfigMap, injecting sensitive credentials from a Secret, defining CPU and memory requests and limits, and attaching a custom ServiceAccount.

This lab uses a **tutorial reference** format. Each step gives you a goal and points to the section of the [Container Configuration](../../../openshift/configuration/index.md), [Config Maps](../../../openshift/configuration/config-map/index.md), [Secrets](../../../openshift/configuration/secrets/index.md), [Service Accounts](../../../openshift/configuration/service-accounts/index.md), or [Projects/Namespaces](../../../openshift/core-concepts/namespaces-projects/index.md) documentation that shows the command or syntax you need. **You build the manifests and commands yourself** by adapting the documentation examples to this lab. When the documentation doesn't cover something, the step gives you the command and explains what it does.

!!! tip "Keep the documentation open"
    Open [Config Maps](../../../openshift/configuration/config-map/index.md), [Secrets](../../../openshift/configuration/secrets/index.md), and [Container Configuration](../../../openshift/configuration/index.md) in a second tab before you start.

## Prerequisites

- You're logged in to an OpenShift cluster with `oc` (or an upstream Kubernetes cluster with `kubectl`). See [Lab Environments](../../../lab-environments.md).
- A terminal and a text editor.

## Part 1: Create a project

1. Create a new project named `lab6` and switch to it.
    - **Reference:** [Projects/Namespaces, References](../../../openshift/core-concepts/namespaces-projects/index.md#references). Create `lab6` instead of `dev`.
    - **Check:** `oc project` shows `Using project "lab6"`.

## Part 2: Create supporting configuration and identity resources

2. Create a ConfigMap named `yoda-service-config` with configuration data stored under key `yoda.cfg`.
    - **Reference:** [Config Maps, References](../../../openshift/configuration/config-map/index.md#references) shows how to create a ConfigMap. Store the following properties under `yoda.cfg`:

    ```properties
    yoda.baby.power=100000000
    yoda.strength=10
    ```

    - **Check:** `oc get configmap yoda-service-config` lists the ConfigMap.

3. Create an Opaque Secret named `yoda-db-password` storing the password `0penSh1ftRul3s!` under key `password`.
    - **Reference:** [Secrets, References](../../../openshift/configuration/secrets/index.md#references) shows creating Secrets with `stringData` or `oc create secret generic`.
    - **Check:** `oc get secret yoda-db-password` lists the Secret.

4. Create a ServiceAccount named `yoda-svc`.
    - **Reference:** [Service Accounts, References](../../../openshift/configuration/service-accounts/index.md#references) shows `oc create sa <name>` and the ServiceAccount YAML structure.
    - **Check:** `oc get sa yoda-svc` lists the ServiceAccount.

## Part 3: Create the configured Pod

Create a Pod manifest named `yoda-service-pod.yaml` meeting these specifications:

- Pod name: `yoda-service`
- ServiceAccount: `yoda-svc`
- Image: `quay.io/nginx/nginx-unprivileged:1.29`
- Environment variable `DB_PASSWORD` populated from `yoda-db-password` key `password`
- Compute resources: requests `cpu: 250m`, `memory: 64Mi`; limits `cpu: 500m`, `memory: 128Mi`
- ConfigMap `yoda-service-config` mounted as a volume at `/etc/yoda-service`

5. Write and apply `yoda-service-pod.yaml`.
    - **Reference:** [Container Configuration, Resource Requirements](../../../openshift/configuration/index.md#resource-requirements) shows `resources.requests` and `limits`. [Secrets, References](../../../openshift/configuration/secrets/index.md#references) shows `valueFrom.secretKeyRef`. [Config Maps, References](../../../openshift/configuration/config-map/index.md#references) shows volume mounting. [Service Accounts, References](../../../openshift/configuration/service-accounts/index.md#references) shows `spec.serviceAccountName`.
    - **Check:** `oc apply -f yoda-service-pod.yaml` creates the Pod, and `oc get pod yoda-service` reaches status `Running`.

## Part 4: Verify the Pod configuration

6. Verify that the configuration file is mounted at `/etc/yoda-service/yoda.cfg`.
    - **Reference:** [Pod Basics, References](../../../openshift/pods/index.md#references). Run `cat` inside the Pod:

    ```bash
    oc exec yoda-service -- cat /etc/yoda-service/yoda.cfg
    ```

    - **Check:** output prints `yoda.baby.power=100000000` and `yoda.strength=10`.

7. Verify that `DB_PASSWORD` is injected as an environment variable from the Secret.
    - **Reference:** [Pod Basics, References](../../../openshift/pods/index.md#references). Run `printenv` inside the Pod:

    ```bash
    oc exec yoda-service -- printenv DB_PASSWORD
    ```

    - **Check:** output prints `0penSh1ftRul3s!`.

8. Verify the ServiceAccount and QoS class assigned to the Pod.
    - **Reference:** [Service Accounts, References](../../../openshift/configuration/service-accounts/index.md#references). Use `jsonpath` to inspect fields:

    ```bash
    oc get pod yoda-service -o jsonpath='{.spec.serviceAccountName}{"\n"}'
    oc get pod yoda-service -o jsonpath='{.spec.containers[0].resources}{"\n"}{.status.qosClass}{"\n"}'
    ```

    - **Check:** output shows `yoda-svc` for serviceAccountName, requests/limits dictionary, and QoS class `Burstable`.

## Cleanup

Delete the `lab6` project and all resources created in this lab.

- **Reference:** [Projects/Namespaces, References](../../../openshift/core-concepts/namespaces-projects/index.md#references) shows project management. To delete the project:

```bash
oc delete project lab6
```

| Part | What it does |
| --- | --- |
| `oc delete project lab6` | Deletes the `lab6` project and all resources inside it |

## What you learned

- [x] Mounting file configurations from ConfigMaps into specific container filesystem paths
- [x] Safely injecting sensitive secrets into environment variables using `secretKeyRef`
- [x] Defining compute resource requests and limits to establish Burstable QoS profiles
- [x] Binding Pod runtime security and API identity to dedicated ServiceAccounts
