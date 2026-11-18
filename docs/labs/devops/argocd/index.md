---
tags:
  - Lab
  - GitOps
  - Argo CD
  - OpenShift
  - Intermediate
---

# Argo CD Lab: Tutorial Reference

<span class="lab-badge">60 min</span> <span class="lab-badge">Intermediate</span>

In this lab you'll practice **GitOps** with **Red Hat OpenShift GitOps**, Red Hat's supported distribution of [Argo CD](https://argo-cd.readthedocs.io/). You'll deploy the bootcamp's `greeting` app from manifests in Git, watch Argo CD correct drift in the cluster, and roll out changes by committing to Git instead of running `oc`.

This lab uses a **tutorial reference** format. Each step gives you a goal and points to the section of the [Argo CD Overview](../../../devops/argocd/index.md) or the OpenShift pages ([Projects/Namespaces](../../../openshift/core-concepts/namespaces-projects/index.md), [Deployments](../../../openshift/deployments/index.md), [Pods](../../../openshift/pods/index.md), [Routes](../../../openshift/services-networking/routes.md)) that show the command you need. **You build the command yourself** by adapting the example to this lab. When those pages don't cover something, such as the `argocd` CLI, the step gives you the command and explains what it does.

!!! tip "Keep the references open"
    Open [What is GitOps?](../../../devops/gitops.md) and the [Deployments](../../../openshift/deployments/index.md) page in other tabs before you start. The Deployments page uses `my-deployment` where this lab uses `greeting`.

## Prerequisites

- An OpenShift 4.16+ cluster with the **Red Hat OpenShift GitOps** operator installed. If it isn't installed and you're a cluster administrator, see [Install the operator](#install-the-operator).
- The `oc` CLI, logged in to the cluster.
- The `argocd` CLI. Install it with `brew install argocd` on macOS, or download it from the [Argo CD releases](https://github.com/argoproj/argo-cd/releases/latest).
- A GitHub account.

### Install the operator

Skip this section if OpenShift GitOps is already installed. Check with `oc get argocd -n openshift-gitops`.

Install the **Red Hat OpenShift GitOps** operator from the web console or with a `Subscription`.

- **Reference:** [How to use Operators, Installing via the Web Console](../../../openshift/operators/operatorUsage.md#installing-via-the-web-console) and [Installing via the CLI](../../../openshift/operators/operatorUsage.md#installing-via-the-cli). Search for **Red Hat OpenShift GitOps** in the console. For the CLI, set the Subscription's `name` to `openshift-gitops-operator`, `channel` to `latest`, `source` to `redhat-operators` and `sourceNamespace` to `openshift-marketplace`.
- **Check:** the operator status shows **Succeeded**.

The operator creates a ready-to-use Argo CD instance in the `openshift-gitops` namespace. Look at its pods and at the Route to its UI.

- **Reference:** [Pods, References](../../../openshift/pods/index.md#references) shows `oc get pods`. [Routes, Commands](../../../openshift/services-networking/routes.md#commands) shows `oc get route`. Add `-n openshift-gitops` to both, and get the Route `openshift-gitops-server`.
- **Check:** the pods are `Running`, and the Route has a host name.

!!! tip "Using plain Kubernetes?"
    Install upstream Argo CD with `kubectl create namespace argocd && kubectl apply -n argocd --server-side -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml`. Everywhere below, use the `argocd` namespace instead of `openshift-gitops`, and skip the `route.yaml` manifest and the namespace label.

## Setup

1. Fork the bootcamp repository to your GitHub account, then clone your fork and look at the `sample-app/gitops` directory.
    - **Reference:** [What is GitOps?, Getting Started with GitOps](../../../devops/gitops.md#getting-started-with-gitops) explains why Git must hold the manifests. [Fork the bootcamp repository](https://github.com/ibm-cloud-architecture/cloud-native-bootcamp/fork) on GitHub. Git isn't covered in the tutorials, so use these commands to clone it:

    ```bash
    export GITHUB_USER=<your-github-username>
    git clone https://github.com/${GITHUB_USER}/cloud-native-bootcamp.git
    cd cloud-native-bootcamp
    ls sample-app/gitops
    ```

    | Part | What it does |
    | --- | --- |
    | `export GITHUB_USER=...` | Stores your username in a variable that later commands reuse |
    | `git clone <url>` | Copies your fork to your machine |
    | `ls sample-app/gitops` | Lists the manifests Argo CD will deploy |

    ```text title="Expected output"
    deployment.yaml  route.yaml  service.yaml
    ```

    - **Check:** you see the three files. They're the **desired state** of the app: a Deployment of the `greeting` image, a Service, and a TLS Route.

2. Create a project named `gitops-lab`. Then label its namespace with `argocd.argoproj.io/managed-by=openshift-gitops`.
    - **Reference:** [Projects/Namespaces, References](../../../openshift/core-concepts/namespaces-projects/index.md#references) shows `oc new-project`. Create `gitops-lab` instead of `dev`. It doesn't cover labelling a namespace, so use this command:

    ```bash
    oc label namespace gitops-lab argocd.argoproj.io/managed-by=openshift-gitops
    ```

    | Part | What it does |
    | --- | --- |
    | `oc label namespace gitops-lab` | Adds a label to the namespace behind the `gitops-lab` project |
    | `argocd.argoproj.io/managed-by=openshift-gitops` | Lets the default Argo CD instance in `openshift-gitops` manage resources in this namespace |

    - **Check:** `oc get namespace gitops-lab --show-labels` lists the label. Without it, syncs fail with a `forbidden` error.

3. Log in to Argo CD.
    - **Check:** you see the Argo CD **Applications** page, or `argocd account get-user-info` prints `Logged In: true`.

    === "Web console"

        Open the Argo CD URL and choose **Log in via OpenShift**. Get the URL from the Route `openshift-gitops-server` with [Routes, Commands](../../../openshift/services-networking/routes.md#commands). You can also use the application launcher (the grid icon) in the OpenShift console and select **Cluster Argo CD**.

    === "CLI"

        The `argocd` CLI isn't covered in the tutorials, so use these commands:

        ```bash
        ARGOCD_SERVER=$(oc get route openshift-gitops-server -n openshift-gitops -o jsonpath='{.spec.host}')
        ARGOCD_PASSWORD=$(oc get secret openshift-gitops-cluster -n openshift-gitops -o jsonpath='{.data.admin\.password}' | base64 -d)
        argocd login "${ARGOCD_SERVER}" --username admin --password "${ARGOCD_PASSWORD}" --grpc-web --insecure
        ```

        | Part | What it does |
        | --- | --- |
        | `ARGOCD_SERVER=$(...)` | Stores the Route's host name |
        | `ARGOCD_PASSWORD=$(...)` | Reads the admin password from the `openshift-gitops-cluster` Secret and decodes it from base64. This needs cluster-admin rights |
        | `argocd login` | Signs in to the Argo CD API server |
        | `--grpc-web` | Sends API calls over HTTP/1.1, which the OpenShift Route supports |
        | `--insecure` | Accepts a self-signed certificate, as OpenShift Local uses. Leave it out when the cluster has a trusted certificate |

## Part 1: Create an Application

An Argo CD **Application** links a path in a Git repository to a namespace in a cluster. Read [How Argo CD Works](../../../devops/argocd/index.md#how-argo-cd-works) first.

### Create greeting-app.yaml

An Application that watches `sample-app/gitops` in your fork and deploys it to `gitops-lab`. Replace `GITHUB_USER` with your GitHub username in the next step.

```yaml title="greeting-app.yaml"
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: greeting
  namespace: openshift-gitops
spec:
  project: default
  source:
    repoURL: https://github.com/GITHUB_USER/cloud-native-bootcamp.git
    targetRevision: main
    path: sample-app/gitops
  destination:
    server: https://kubernetes.default.svc
    namespace: gitops-lab
```

- `kind: Application` is a resource that the OpenShift GitOps operator added to the cluster. It lives in the `openshift-gitops` namespace, where the Argo CD instance runs.
- `project: default` is the Argo CD project, which sets what the Application may deploy and where. It isn't an OpenShift project.
- `source` says where the desired state is. `repoURL` is the Git repository, `targetRevision` is the branch, tag or commit, and `path` is the directory with the manifests.
- `destination` says where to deploy. `https://kubernetes.default.svc` means the cluster Argo CD runs in. `namespace` is the target namespace.
- There's no `syncPolicy`, so Argo CD only reports differences and you sync by hand.

4. Put your GitHub username in `greeting-app.yaml` and create the Application.
    - **Reference:** [Deployments, References](../../../openshift/deployments/index.md#references) shows `oc apply -f`. It doesn't cover editing a file in place, so use this command to set your username:

    ```bash
    sed -i.bak "s/GITHUB_USER/${GITHUB_USER}/" greeting-app.yaml
    ```

    | Part | What it does |
    | --- | --- |
    | `sed -i.bak` | Edits the file in place and keeps the original as `greeting-app.yaml.bak` |
    | `"s/GITHUB_USER/${GITHUB_USER}/"` | Replaces the text `GITHUB_USER` with the value of the variable |

    - **Check:** `grep repoURL greeting-app.yaml` shows your username, and `oc get application greeting -n openshift-gitops` lists the Application.

5. Check the Application's status in the UI or with the CLI.
    - **Reference:** [How Argo CD Works](../../../devops/argocd/index.md#how-argo-cd-works), steps 3 and 4, explains the comparison between Git and the cluster. It doesn't cover the CLI, so use this command:

    ```bash
    argocd app get openshift-gitops/greeting
    ```

    `argocd app get` prints the Application's source, destination, sync status, health status and resources. `openshift-gitops/greeting` is `<namespace>/<name>`.

    - **Check:** `Sync Status` is `OutOfSync` and `Health Status` is `Missing`. Argo CD compared Git with the cluster and found that none of the resources exist yet. With no sync policy, it only reports the difference.

6. **Sync** the Application to make the cluster match Git, and wait until it is healthy.
    - **Reference:** [How Argo CD Works](../../../devops/argocd/index.md#how-argo-cd-works), step 4, describes syncing. It doesn't cover the CLI, so use these commands, or click **Sync > Synchronize** in the UI:

    ```bash
    argocd app sync openshift-gitops/greeting
    argocd app wait openshift-gitops/greeting --health
    ```

    | Part | What it does |
    | --- | --- |
    | `argocd app sync` | Applies the manifests from Git to the cluster |
    | `argocd app wait --health` | Blocks until every resource of the Application is healthy |

    - **Check:** the sync ends with `Sync Status: Synced` and `Health Status: Healthy`.

7. List the Deployment, Service and Route in `gitops-lab`, then call the app with the name `GitOps`.
    - **Reference:** [Deployments, References](../../../openshift/deployments/index.md#references) shows `oc get`. [Services, References](../../../openshift/services-networking/services.md#references) shows `oc get svc`. [Routes, Commands](../../../openshift/services-networking/routes.md#commands) shows how to get a Route's URL. Select the project with `-n gitops-lab`. The Route uses TLS, so call it over `https` with `curl -sk`.
    - **Check:** the response is `{"message":"Deployed by Argo CD","name":"GitOps"}`.

## Part 2: Automated sync and self-healing

8. Turn on automated sync with **prune** (delete resources that were removed from Git) and **self-heal** (undo manual changes in the cluster).
    - **Reference:** [Key Features](../../../devops/argocd/index.md#key-features) describes automated deployment. It doesn't cover sync policies, so use this command:

    ```bash
    argocd app set openshift-gitops/greeting --sync-policy automated --auto-prune --self-heal
    ```

    | Part | What it does |
    | --- | --- |
    | `argocd app set` | Changes the Application's settings |
    | `--sync-policy automated` | Syncs without being asked whenever Git and the cluster differ |
    | `--auto-prune` | Deletes cluster resources that were removed from Git |
    | `--self-heal` | Reverts changes made directly in the cluster |

    Declaratively, the same settings are this block in the Application's `spec`:

    ```yaml
    syncPolicy:
      automated:
        prune: true
        selfHeal: true
    ```

    - **Check:** `argocd app get openshift-gitops/greeting` shows `Sync Policy: Automated (Prune)`.

9. Simulate someone "fixing" production by hand: scale the `greeting` Deployment to 3 replicas, then watch it.
    - **Reference:** [Deployments, References](../../../openshift/deployments/index.md#references) shows `oc scale` and `oc get deployment`. Add `-n gitops-lab`, and add `-w` to watch.
    - **Check:** within a few seconds, Argo CD notices that the cluster has **drifted** from Git and scales the Deployment back to 1 replica. Press ++ctrl+c++ to stop watching.

## Part 3: Change the app through Git

In GitOps, Git is the only way to change what runs.

10. In your clone, edit `sample-app/gitops/deployment.yaml`. Set `replicas: 2` and change the `GREETING` value to a message of your own.
    - **Reference:** [Deployments, References](../../../openshift/deployments/index.md#references) shows the `replicas` field of a Deployment. [Container Configuration, Environment Variable](../../../openshift/configuration/index.md#environment-variable) explains `env`.
    - **Check:** `git diff` shows your two changes.

11. Commit and push the change.
    - **Reference:** [What is GitOps?, How GitOps Works](../../../devops/gitops.md#how-gitops-works) describes changing the system through Git. Git isn't covered in the tutorials, so use these commands:

    ```bash
    git add sample-app/gitops/deployment.yaml
    git commit -m "Scale greeting to 2 replicas and update message"
    git push
    ```

    | Part | What it does |
    | --- | --- |
    | `git add <file>` | Stages the file for the next commit |
    | `git commit -m "..."` | Records the staged change with a message |
    | `git push` | Sends the commit to your fork on GitHub |

    - **Check:** the commit appears on your fork on GitHub.

12. Wait for Argo CD to pick up the commit, then check the pods and call the app. Argo CD polls Git every 3 minutes. Click **Refresh** in the UI to check right away.
    - **Reference:** [Pods, References](../../../openshift/pods/index.md#references) shows `oc get pods`. Add `-n gitops-lab -l app=greeting` to select the app's pods. [Routes, Commands](../../../openshift/services-networking/routes.md#commands) shows how to get a Route's URL. It doesn't cover refreshing from the CLI, so use this command instead of the UI button:

    ```bash
    argocd app get openshift-gitops/greeting --refresh
    ```

    `--refresh` makes Argo CD fetch from Git now instead of waiting for the next poll.

    - **Check:** there are 2 `greeting` pods, and the response contains your new message.

13. See the deployment history.
    - **Reference:** [Key Features](../../../devops/argocd/index.md#key-features) mentions rollbacks to any state committed in Git. It doesn't cover the history command, so use this command:

    ```bash
    argocd app history openshift-gitops/greeting
    ```

    `argocd app history` lists every sync with its ID, time and the Git commit it deployed.

    - **Check:** the list shows at least two syncs, each with a different commit.

14. **Roll back the GitOps way**: revert the commit and push. Argo CD brings the cluster back in line with Git.
    - **Reference:** [What is GitOps? vs Traditional CI/CD](../../../devops/gitops.md#gitops-vs-traditional-cicd) says that a rollback in GitOps is a Git revert. Git isn't covered in the tutorials, so use these commands:

    ```bash
    git revert --no-edit HEAD
    git push
    ```

    | Part | What it does |
    | --- | --- |
    | `git revert HEAD` | Creates a new commit that undoes the latest commit. History is kept |
    | `--no-edit` | Uses the default commit message instead of opening an editor |

    - **Check:** after the next sync, `oc get pods -n gitops-lab -l app=greeting` shows 1 pod, and the original message is back.

!!! tip "Deploy your own image"
    Change the `image` in `deployment.yaml` to the `quay.io/<your-username>/greeting:v1.0.0` image you pushed in the [Containers Lab](../../containers/lab1/index.md), then commit and push. This is how CI and CD connect: a pipeline builds the image, and a commit to the GitOps repository deploys it. The [Tekton Lab](../tekton/index.md) builds that pipeline.

## Part 4: Prune

15. Delete `sample-app/gitops/route.yaml` from your fork, then commit and push.
    - **Reference:** use the `git add`, `git commit` and `git push` commands from step 11. Use `git rm sample-app/gitops/route.yaml` in place of `git add`.
    - **Check:** the commit appears on GitHub without the file.

16. After the next sync, check that the Route is gone from the cluster.
    - **Reference:** [Routes, Commands](../../../openshift/services-networking/routes.md#commands) shows `oc get route`. Add `-n gitops-lab`.
    - **Check:** `oc get route -n gitops-lab` lists no `greeting` Route, because `prune` is on.

17. Restore the Route by reverting the commit.
    - **Reference:** use the `git revert` and `git push` commands from step 14.
    - **Check:** after the next sync, the `greeting` Route is back.

## Cleanup

Delete the Application, then the `gitops-lab` project.

- **Reference:** [Projects/Namespaces, References](../../../openshift/core-concepts/namespaces-projects/index.md#references) shows project management. It doesn't cover deleting a project or an Application, so use these commands:

```bash
oc delete -f greeting-app.yaml       # deletes the Application only
oc delete project gitops-lab
```

!!! note
    Deleting an Application doesn't delete the resources it deployed, unless the Application has the `resources-finalizer.argocd.argoproj.io` finalizer. Deleting the project removes everything.

## What you learned

- [x] Declaring an Argo CD Application that points at a Git path
- [x] Manual vs. automated sync, self-heal and prune
- [x] Changing and rolling back an application by committing to Git
- [x] How CI (building images) and CD (GitOps) fit together

## Additional resources

- [Red Hat OpenShift GitOps documentation](https://docs.redhat.com/en/documentation/red_hat_openshift_gitops/)
- [Argo CD documentation](https://argo-cd.readthedocs.io/en/stable/)
- [OpenGitOps principles](https://opengitops.dev/)
