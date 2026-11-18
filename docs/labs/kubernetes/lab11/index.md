---
tags:
  - Lab
  - Intermediate
  - Networking
  - OpenShift only
  - Routes
  - Ingress
  - OpenShift
  - Kubernetes
---

# Routes & Ingress Lab

<span class="lab-badge">30 min</span> <span class="lab-badge">Intermediate</span> <span class="lab-badge">OpenShift only</span>

In this lab you'll publish an application running on OpenShift to external clients using native OpenShift **Routes** (unsecured HTTP and edge-terminated TLS) as well as portable Kubernetes **Ingress** resources.

This lab uses a **tutorial reference** format. Each step gives you a goal and points to the section of the [Routes](../../../openshift/services-networking/routes.md), [Ingress](../../../openshift/services-networking/ingress.md), [Services](../../../openshift/services-networking/services.md), or [Projects/Namespaces](../../../openshift/core-concepts/namespaces-projects/index.md) documentation that shows the command or syntax you need. **You build the manifests and commands yourself** by adapting the documentation examples to this lab. When the documentation doesn't cover something, the step gives you the command and explains what it does.

!!! tip "Keep the documentation open"
    Open [Routes](../../../openshift/services-networking/routes.md) and [Ingress](../../../openshift/services-networking/ingress.md) in a second tab before you start.

## Prerequisites

- You're logged in to an OpenShift cluster with `oc`. See [Lab Environments](../../../lab-environments.md).
- A terminal and a text editor.

## Part 1: Deploy and expose the application

1. Create a project named `lab11` and switch to it.
    - **Reference:** [Projects/Namespaces, References](../../../openshift/core-concepts/namespaces-projects/index.md#references). Create `lab11` instead of `dev`.
    - **Check:** `oc project` shows `Using project "lab11"`.

2. Deploy the `web` application and expose it internally as a Service on port `8080`.
    - **Reference:** [Ingress, Commands](../../../openshift/services-networking/ingress.md#references) shows deploying and exposing with `oc create deployment` and `oc expose deployment`.

    ```bash
    oc create deployment web --image=quay.io/nginx/nginx-unprivileged:1.29 --port=8080
    oc expose deployment web --port=8080
    oc rollout status deployment/web
    ```

    - **Check:** `oc get svc web` lists the `web` Service on port `8080`.

## Part 2: Publish using OpenShift Routes

3. Create an HTTP Route named `web` for the `web` Service and verify external access via `curl`.
    - **Reference:** [Routes, Commands](../../../openshift/services-networking/routes.md#commands) shows creating a Route with `oc expose service`.
    - Test the route with:

    ```bash
    curl -s "http://$(oc get route web -o jsonpath='{.spec.host}')" | grep "<title>"
    ```

    - **Check:** `curl` returns `<title>Welcome to nginx!</title>`.

4. Create an edge-terminated TLS Route named `web-secure` with HTTP-to-HTTPS redirect for the `web` Service.
    - **Reference:** [Routes, Commands](../../../openshift/services-networking/routes.md#commands) shows `oc create route edge <name> --service=<service> --insecure-policy=Redirect`.
    - Verify that HTTP returns status code `302` / `301` redirect and HTTPS serves the web page:

    ```bash
    curl -s -o /dev/null -w '%{http_code}\n' "http://$(oc get route web-secure -o jsonpath='{.spec.host}')"
    curl -sk "https://$(oc get route web-secure -o jsonpath='{.spec.host}')" | grep "<title>"
    ```

    - **Check:** plain HTTP request returns a `302` redirect status code, and HTTPS (`-sk`) returns `<title>Welcome to nginx!</title>`.

## Part 3: Publish using Kubernetes Ingress

5. Create an Ingress resource named `web-ingress` targeting host `web-ingress-lab11.<apps-domain>`.
    - **Reference:** [Ingress, References](../../../openshift/services-networking/ingress.md#references) shows the `networking.k8s.io/v1` `Ingress` schema. Determine your cluster's apps domain and apply the Ingress:

    ```bash
    APPS_DOMAIN=$(oc get route web -o jsonpath='{.spec.host}' | cut -d. -f2-)
    oc apply -f - <<EOF
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: web-ingress
    spec:
      rules:
        - host: web-ingress-lab11.${APPS_DOMAIN}
          http:
            paths:
              - path: /
                pathType: Prefix
                backend:
                  service:
                    name: web
                    port:
                      number: 8080
    EOF
    ```

    - **Check:** `oc get ingress web-ingress` shows the configured host, and `oc get routes` displays an auto-generated Route matching the Ingress.

6. Send an HTTP request to the Ingress host and confirm it serves traffic through the generated Route.
    - **Reference:** [Ingress, Commands](../../../openshift/services-networking/ingress.md#references).

    ```bash
    curl -s "http://web-ingress-lab11.${APPS_DOMAIN}" | grep "<title>"
    ```

    - **Check:** output returns `<title>Welcome to nginx!</title>`.

## Cleanup

Delete the `lab11` project and all resources created in this lab.

- **Reference:** [Projects/Namespaces, References](../../../openshift/core-concepts/namespaces-projects/index.md#references) shows project management. To delete the project:

```bash
oc delete project lab11
```

| Part | What it does |
| --- | --- |
| `oc delete project lab11` | Deletes the `lab11` project, Deployment, Service, Routes, and Ingress |

## What you learned

- [x] Exposing internal cluster Services through OpenShift HAProxy router using Routes
- [x] Enforcing HTTPS with edge TLS termination and HTTP redirect policies
- [x] Deploying portable Kubernetes Ingress definitions and observing automatic OpenShift Route synthesis
- [x] Extracting cluster domain metadata dynamically with `jsonpath`
