---
tags:
  - Lab
  - Beginner
  - Pods
  - OpenShift
  - Kubernetes
---

# Pod Creation Lab

<span class="lab-badge">15 min</span> <span class="lab-badge">Beginner</span>

In this lab you'll write a Pod manifest, configure labels, container ports, commands, and arguments, deploy it to an OpenShift project, and verify it serves traffic.

This lab uses a **tutorial reference** format. Each step gives you a goal and points to the section of the [Pod Basics](../../../openshift/pods/index.md), [Projects/Namespaces](../../../openshift/core-concepts/namespaces-projects/index.md), [Labels & Selectors](../../../openshift/pods/tagging/index.md), or [Container Configuration](../../../openshift/configuration/index.md) documentation that shows the command or syntax you need. **You build the manifest and commands yourself** by adapting the documentation examples to this lab. When the documentation doesn't cover something, the step gives you the command and explains what it does.

!!! tip "Keep the documentation open"
    Open [Pod Basics](../../../openshift/pods/index.md) and [Container Configuration](../../../openshift/configuration/index.md) in a second tab before you start. This lab creates a Pod named `yoda-service` with image `quay.io/nginx/nginx-unprivileged:1.29` where the documentation uses `myapp-pod` and `busybox`.

## Prerequisites

- You're logged in to an OpenShift cluster with `oc` (or an upstream Kubernetes cluster with `kubectl`). See [Lab Environments](../../../lab-environments.md).
- A terminal and a text editor.

## Part 1: Create a project

1. Create a new project (namespace) named `web` and switch to it.
    - **Reference:** [Projects/Namespaces, References](../../../openshift/core-concepts/namespaces-projects/index.md#references). Create a project named `web` instead of `dev`.
    - **Check:** `oc project` shows `Using project "web"`.

## Part 2: Write the Pod manifest

Create a file named `yoda-service-pod.yaml` in your working directory. You will assemble the manifest using the requirements below.

2. Define the Pod metadata with name `yoda-service`, namespace `web`, and label `app: yoda`.
    - **Reference:** [Pod Basics, References](../../../openshift/pods/index.md#references) shows the basic Pod skeleton (`apiVersion`, `kind`, `metadata.name`). [Projects/Namespaces, References](../../../openshift/core-concepts/namespaces-projects/index.md#references) shows specifying `metadata.namespace`. [Labels & Selectors, References](../../../openshift/pods/tagging/index.md#references) shows adding `metadata.labels`.
    - **Check:** your manifest defines `kind: Pod` with `name: yoda-service`, `namespace: web`, and `labels.app: yoda`.

3. Define a container named `nginx` using the image `quay.io/nginx/nginx-unprivileged:1.29`, exposing container port `8080`.
    - **Reference:** [Container Configuration, Ports](../../../openshift/configuration/index.md#ports) shows how to specify `ports` and `containerPort`.
    - **Check:** your container spec includes `image: quay.io/nginx/nginx-unprivileged:1.29` and `containerPort: 8080`.

4. Override the container entrypoint and command to run `nginx` with arguments `-g` and `daemon off;` so it runs in the foreground.
    - **Reference:** [Container Configuration, Command and Argument](../../../openshift/configuration/index.md#command-and-argument) shows `command` (overrides Dockerfile `ENTRYPOINT`) and `args` (overrides Dockerfile `CMD`).
    - **Check:** your container spec includes `command: ["nginx"]` and `args: ["-g", "daemon off;"]`.

!!! info "Why port 8080 and not 80?"
    OpenShift runs containers as a random, non-root user, and non-root processes can't bind to privileged ports below 1024. OpenShift-friendly images, such as `nginx-unprivileged` or Red Hat UBI images, listen on an unprivileged high port such as 8080 instead.

## Part 3: Deploy and verify the Pod

5. Create the Pod in your cluster from `yoda-service-pod.yaml`.
    - **Reference:** [Pod Basics, References](../../../openshift/pods/index.md#references) shows how to create a Pod using `oc apply -f`.
    - **Check:** the command outputs `pod/yoda-service created`.

6. Verify that the Pod reaches the `Running` state and all containers are ready (`1/1`).
    - **Reference:** [Pod Basics, References](../../../openshift/pods/index.md#references) shows how to list pods (`oc get pods`) and view pod descriptions (`oc describe pod`).
    - **Check:** `oc get pods -n web` shows `yoda-service` with status `Running` and `1/1` ready.

7. Forward local port `8080` to the Pod and verify NGINX serves its welcome page.
    - **Reference:** [Debugging Applications, A broken application to practice on](../../../openshift/pods/troubleshooting/index.md#a-broken-application-to-practice-on) shows port forwarding to a service. The documentation doesn't cover port forwarding directly to a Pod, so run this command in one terminal:

    ```bash
    oc port-forward pod/yoda-service 8080:8080 -n web
    ```

    | Part | What it does |
    | --- | --- |
    | `oc port-forward pod/yoda-service` | Forwards traffic from your local machine to the `yoda-service` Pod |
    | `8080:8080` | Maps local port 8080 to container port 8080 |
    | `-n web` | Directs the command to the `web` project |

    In a second terminal, send an HTTP request:

    ```bash
    curl -s localhost:8080 | grep "<title>"
    ```

    ```text title="Expected output"
    <title>Welcome to nginx!</title>
    ```

    - **Check:** `curl` returns `<title>Welcome to nginx!</title>`.

## Cleanup

Delete the `web` project to remove all resources created in this lab.

- **Reference:** [Projects/Namespaces, References](../../../openshift/core-concepts/namespaces-projects/index.md#references) shows project management. The documentation doesn't cover project deletion, so use this command:

```bash
oc delete project web
```

| Part | What it does |
| --- | --- |
| `oc delete project web` | Deletes the `web` project and all resources inside it |

## What you learned

- [x] Creating and managing OpenShift projects
- [x] Writing a Pod manifest with custom labels, container ports, commands, and arguments
- [x] Deploying and inspecting Pods using the `oc` CLI
- [x] Verifying containerized services using port forwarding
