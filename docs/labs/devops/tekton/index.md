---
tags:
  - Lab
  - CI/CD
  - Tekton
  - OpenShift
  - Intermediate
---

# Tekton Lab: Tutorial Reference

<span class="lab-badge">60 min</span> <span class="lab-badge">Intermediate</span>

In this lab you'll use **Red Hat OpenShift Pipelines**, Red Hat's supported distribution of [Tekton](https://tekton.dev/), to build a continuous integration pipeline. The pipeline clones the bootcamp's sample app from Git, builds a container image, pushes it to OpenShift's internal registry, and deploys it.

This lab uses a **tutorial reference** format. Each step gives you a goal and points to the section of the [Tekton Overview](../../../devops/tekton/index.md) or the OpenShift pages ([Projects/Namespaces](../../../openshift/core-concepts/namespaces-projects/index.md), [Deployments](../../../openshift/deployments/index.md), [Pods](../../../openshift/pods/index.md), [Routes](../../../openshift/services-networking/routes.md)) that show the command you need. **You build the command yourself** by adapting the example to this lab. When those pages don't cover something, such as the `tkn` CLI, the step gives you the command and explains what it does.

!!! tip "Keep the references open"
    Open the [Projects/Namespaces](../../../openshift/core-concepts/namespaces-projects/index.md) and [Pods](../../../openshift/pods/index.md) pages in other tabs before you start. They use `dev` and `myapp-pod` where this lab uses `tekton-lab` and the `greeting` app.

## Prerequisites

- An OpenShift 4.16+ cluster with the **Red Hat OpenShift Pipelines** operator installed. If it isn't installed and you're a cluster administrator, see [Install the operator](#install-the-operator). OpenShift Local and the Developer Sandbox can both run this lab.
- The `oc` CLI, logged in to the cluster.
- The `tkn` CLI:

    === "macOS"

        ```bash
        brew install tektoncd-cli
        ```

    === "Linux"

        Download the latest `tkn` archive for your architecture from the [Tekton CLI releases](https://github.com/tektoncd/cli/releases/latest), extract it, and move `tkn` onto your `PATH`. You can also download `tkn` from your cluster's web console, under the **?** menu > **Command Line Tools**.

    === "Windows"

        With [Chocolatey](https://community.chocolatey.org/packages/tektoncd-cli):

        ```powershell
        choco install tektoncd-cli --confirm
        ```

    Check it works with `tkn version`.

### Install the operator

Skip this section if OpenShift Pipelines is already installed. Check with `oc get csv -n openshift-operators | grep pipelines`.

Install the **Red Hat OpenShift Pipelines** operator from the web console or with a `Subscription`.

- **Reference:** [How to use Operators, Installing via the Web Console](../../../openshift/operators/operatorUsage.md#installing-via-the-web-console) and [Installing via the CLI](../../../openshift/operators/operatorUsage.md#installing-via-the-cli). Search for **Red Hat OpenShift Pipelines** in the console. For the CLI, set the Subscription's `name` to `openshift-pipelines-operator-rh`, `channel` to `latest`, `source` to `redhat-operators` and `sourceNamespace` to `openshift-marketplace`.
- **Check:** the operator status shows **Succeeded**, and `oc get pods -n openshift-pipelines` lists running Tekton pods.

## Tekton concepts

Read the [Tekton Overview](../../../devops/tekton/index.md#tekton-overview) first. These are the resources you'll use:

| Resource | What it is |
| --- | --- |
| **Step** | A container that runs one command or script |
| **Task** | An ordered list of steps. Each Task run is one pod. |
| **Pipeline** | A graph of Tasks that run in sequence or in parallel |
| **TaskRun / PipelineRun** | One execution of a Task or Pipeline, with its parameter values, workspaces and logs |
| **Workspace** | Storage shared between Tasks, for example the cloned source code |
| **Resolver** | Fetches Task and Pipeline definitions from the cluster, a Git repo, a bundle or Artifact Hub |

## Setup

1. Create a project named `tekton-lab` and switch to it.
    - **Reference:** [Projects/Namespaces, References](../../../openshift/core-concepts/namespaces-projects/index.md#references). Create `tekton-lab` instead of `dev`.
    - **Check:** `oc project` shows `tekton-lab`.

OpenShift Pipelines automatically creates a `pipeline` ServiceAccount in every project. PipelineRuns use it, and it has permission to push to the project's image streams and to create Deployments.

## Part 1: Your first Task

### Create hello-task.yaml

A Task with one step that prints a greeting. It takes one parameter, `name`.

```yaml title="hello-task.yaml"
apiVersion: tekton.dev/v1
kind: Task
metadata:
  name: hello
spec:
  params:
    - name: name
      type: string
      default: Padawan
  steps:
    - name: say-hello
      image: registry.access.redhat.com/ubi9/ubi-minimal
      script: |
        #!/usr/bin/env bash
        echo "Hello, $(params.name)! Running in pod ${HOSTNAME}."
```

- `kind: Task` is a Tekton resource that the OpenShift Pipelines operator added to the cluster.
- `params` declares the inputs of the Task. `default` is used when the caller doesn't pass a value.
- `steps` is the ordered list of containers. Each step has its own `image`, and all steps of a Task run in one pod.
- `script` is written into the container and run. `$(params.name)` is replaced with the parameter value before the step starts.

2. Create the Task from `hello-task.yaml`, then list the Tasks in your project.
    - **Reference:** [Deployments, References](../../../openshift/deployments/index.md#references) shows `oc apply -f`. It doesn't cover listing Tekton resources, so use this command to list the Tasks:

    ```bash
    tkn task list
    ```

    `tkn task list` prints the Tasks in the current project, with their age.

    - **Check:** `hello` is in the list.

3. Run the `hello` Task, override the `name` parameter with `Jedi`, and stream the logs.
    - **Reference:** [Tekton Overview](../../../devops/tekton/index.md#tekton-overview) introduces the `tkn` CLI. It doesn't cover running a Task, so use this command:

    ```bash
    tkn task start hello -p name=Jedi --showlog
    ```

    | Part | What it does |
    | --- | --- |
    | `tkn task start hello` | Creates a **TaskRun** for the Task `hello` |
    | `-p name=Jedi` | Sets the parameter `name`, which replaces the default `Padawan` |
    | `--showlog` | Streams the logs of the TaskRun until it finishes |

    ```text title="Expected output"
    TaskRun started: hello-run-8xk2p
    Waiting for logs to be available...
    [say-hello] Hello, Jedi! Running in pod hello-run-8xk2p-pod.
    ```

    - **Check:** the log line says `Hello, Jedi!`.

4. `tkn` created a **TaskRun** object for you. List the TaskRuns, then look at the pod that ran the step.
    - **Reference:** [Pods, References](../../../openshift/pods/index.md#references) shows `oc get pods`. It doesn't cover TaskRuns, so use this command to list them: `tkn taskrun list`
    - **Check:** the TaskRun has status `Succeeded`, and the pod `hello-run-8xk2p-pod` (your suffix differs) has status `Completed`.

## Part 2: A build-and-deploy Pipeline

Instead of writing every Task yourself, reuse the Tasks that OpenShift Pipelines ships in the `openshift-pipelines` namespace. You reference them with the **cluster resolver**.

5. List the Tasks in the `openshift-pipelines` namespace.
    - **Reference:** [Deployments, References](../../../openshift/deployments/index.md#references) shows `oc get <resource> <name>`. List `tasks` instead of a deployment, and select the namespace with `-n openshift-pipelines`.
    - **Check:** the list contains `git-clone`, `buildah` and `openshift-client`.

This lab uses three of them:

| Task | Purpose | Key parameters |
| --- | --- | --- |
| `git-clone` | Clone a Git repository into a workspace | `URL`, `REVISION` |
| `buildah` | Build an image from a Containerfile and push it | `IMAGE`, `DOCKERFILE`, `CONTEXT` |
| `openshift-client` | Run `oc` commands | `SCRIPT` |

!!! note "Coming from ClusterTasks?"
    Older tutorials use `kind: ClusterTask` references. OpenShift Pipelines 1.17 removed ClusterTasks. Use `resolver: cluster`, as shown below.

### Create pipeline.yaml

A Pipeline with three Tasks that run one after another: clone, build, deploy.

```yaml title="pipeline.yaml"
apiVersion: tekton.dev/v1
kind: Pipeline
metadata:
  name: build-and-deploy
spec:
  params:
    - name: git-url
      type: string
      default: https://github.com/ibm-cloud-architecture/cloud-native-bootcamp.git
    - name: git-revision
      type: string
      default: main
    - name: context-dir
      type: string
      description: Directory in the repo that contains the Containerfile
      default: sample-app/greeting
    - name: app-name
      type: string
      default: greeting
  workspaces:
    - name: source
  tasks:
    - name: fetch-source
      taskRef:
        resolver: cluster
        params:
          - { name: kind, value: task }
          - { name: name, value: git-clone }
          - { name: namespace, value: openshift-pipelines }
      params:
        - { name: URL, value: $(params.git-url) }
        - { name: REVISION, value: $(params.git-revision) }
      workspaces:
        - { name: output, workspace: source }

    - name: build-image
      runAfter: [fetch-source]
      taskRef:
        resolver: cluster
        params:
          - { name: kind, value: task }
          - { name: name, value: buildah }
          - { name: namespace, value: openshift-pipelines }
      params:
        - name: IMAGE
          value: image-registry.openshift-image-registry.svc:5000/$(context.pipelineRun.namespace)/$(params.app-name):latest
        - { name: CONTEXT, value: $(params.context-dir) }
        - { name: DOCKERFILE, value: $(params.context-dir)/Containerfile }
      workspaces:
        - { name: source, workspace: source }

    - name: deploy
      runAfter: [build-image]
      taskRef:
        resolver: cluster
        params:
          - { name: kind, value: task }
          - { name: name, value: openshift-client }
          - { name: namespace, value: openshift-pipelines }
      params:
        - name: SCRIPT
          value: |
            IMAGE="$(tasks.build-image.results.IMAGE_URL)@$(tasks.build-image.results.IMAGE_DIGEST)"
            oc get deployment $(params.app-name) || \
              oc create deployment $(params.app-name) --image="${IMAGE}" --port=8080
            oc set image deployment/$(params.app-name) "*=${IMAGE}"
            oc rollout status deployment/$(params.app-name) --timeout=3m
            oc get service $(params.app-name) || oc expose deployment $(params.app-name) --port=8080
            oc get route $(params.app-name) || oc expose service $(params.app-name)
```

- `params` are the inputs of the Pipeline. The defaults point at the bootcamp repository and the `greeting` app.
- `workspaces` declares storage that Tasks share. `fetch-source` writes the cloned code into `source`, and `build-image` reads it.
- `taskRef` with `resolver: cluster` fetches the Task from the `openshift-pipelines` namespace instead of from your project.
- `runAfter` orders the Tasks. Tasks without it run in parallel.
- `$(tasks.build-image.results.IMAGE_DIGEST)` passes a **result** from one Task to another. Deploying by digest guarantees that every build rolls out, even though the tag (`latest`) stays the same.
- `$(context.pipelineRun.namespace)` resolves to the project the pipeline runs in.
- The `deploy` script is idempotent: it creates the Deployment, Service and Route on the first run and only updates the image afterwards.

6. Create the Pipeline from `pipeline.yaml` and look at it.
    - **Reference:** [Deployments, References](../../../openshift/deployments/index.md#references) shows `oc apply -f`. It doesn't cover inspecting a Pipeline, so use this command:

    ```bash
    tkn pipeline describe build-and-deploy
    ```

    `tkn pipeline describe` prints the Pipeline's parameters, workspaces and Tasks, and its most recent runs.

    - **Check:** the output lists the three Tasks `fetch-source`, `build-image` and `deploy`.

7. Start a run of the Pipeline and stream the logs. Give the `source` workspace a new 1 GiB PersistentVolumeClaim for each run, created from a **volumeClaimTemplate**.
    - **Reference:** [Tekton Overview](../../../devops/tekton/index.md#tekton-overview) introduces the `tkn` CLI. It doesn't cover starting a Pipeline or creating a workspace volume, so use one of these:

    === "tkn"

        ```bash
        cat > workspace-template.yaml <<'EOF'
        spec:
          accessModes: [ReadWriteOnce]
          resources:
            requests:
              storage: 1Gi
        EOF

        tkn pipeline start build-and-deploy \
          --use-param-defaults \
          --workspace name=source,volumeClaimTemplateFile=workspace-template.yaml \
          --showlog
        ```

    === "PipelineRun YAML"

        ```yaml title="pipelinerun.yaml"
        apiVersion: tekton.dev/v1
        kind: PipelineRun
        metadata:
          generateName: build-and-deploy-
        spec:
          pipelineRef:
            name: build-and-deploy
          workspaces:
            - name: source
              volumeClaimTemplate:
                spec:
                  accessModes: [ReadWriteOnce]
                  resources:
                    requests:
                      storage: 1Gi
        ```

        ```bash
        oc create -f pipelinerun.yaml
        tkn pipelinerun logs --last -f
        ```

    | Part | What it does |
    | --- | --- |
    | `tkn pipeline start build-and-deploy` | Creates a **PipelineRun** for the Pipeline |
    | `--use-param-defaults` | Uses the default value of every parameter instead of asking for each |
    | `--workspace name=source,volumeClaimTemplateFile=...` | Binds the `source` workspace to a new PersistentVolumeClaim created from the template file for this run |
    | `--showlog` | Streams the logs of the PipelineRun until it finishes |
    | `generateName` | Gives each PipelineRun created from the YAML a unique name (`oc create` is needed because `oc apply` requires a fixed name) |
    | `tkn pipelinerun logs --last -f` | Follows the logs of the most recent PipelineRun |

    - **Check:** the logs show all three Tasks finishing, and `tkn pipelinerun list` shows the run with status `Succeeded`.

8. Call the `greeting` app with the name `Tekton`.
    - **Reference:** [Routes, Commands](../../../openshift/services-networking/routes.md#commands) shows how to get a Route's URL. Get the Route `greeting`, which the pipeline created, and call `/greeting?name=Tekton` over `http` with `curl`.
    - **Check:** the response is JSON containing `"name":"Tekton"`.

## Part 3: The web console

9. In the web console, go to **Pipelines** and select the `tekton-lab` project. Open `build-and-deploy` to see the Tasks as a graph, and open a PipelineRun to see each Task's logs and duration.
    - **Check:** the graph shows `fetch-source`, `build-image` and `deploy` in order.

10. Start another run from **Actions > Start**. Choose **VolumeClaimTemplate** for the `source` workspace.
    - **Check:** a new PipelineRun appears in the list and succeeds.

11. Change the `GREETING` environment variable of the `greeting` Deployment to `Hello from the pipeline`, run the pipeline again, and check whether your setting survives the redeploy. Why does it?
    - **Reference:** [Container Configuration, Environment Variable](../../../openshift/configuration/index.md#environment-variable) explains environment variables. It doesn't cover changing them on a running Deployment, so use this command:

    ```bash
    oc set env deployment/greeting GREETING="Hello from the pipeline"
    ```

    `oc set env` updates the environment of the Deployment's containers and rolls out new pods.

    - **Check:** after the second run, `curl` on the Route still returns your message. Look at the `deploy` script in `pipeline.yaml` to find out why.

## Part 4 (optional): Build your own fork on every push

To run the pipeline automatically when code changes, your Git provider has to trigger it.

12. Fork [the bootcamp repository](https://github.com/ibm-cloud-architecture/cloud-native-bootcamp) on GitHub.
    - **Check:** the fork appears in your GitHub account.

### Create triggers.yaml

Three Tekton Triggers resources that receive GitHub push webhooks and start a PipelineRun for the pushed commit.

```yaml title="triggers.yaml"
apiVersion: triggers.tekton.dev/v1beta1
kind: TriggerBinding
metadata:
  name: github-push
spec:
  params:
    - name: git-url
      value: $(body.repository.clone_url)
    - name: git-revision
      value: $(body.after)
---
apiVersion: triggers.tekton.dev/v1beta1
kind: TriggerTemplate
metadata:
  name: build-and-deploy
spec:
  params:
    - name: git-url
    - name: git-revision
  resourcetemplates:
    - apiVersion: tekton.dev/v1
      kind: PipelineRun
      metadata:
        generateName: build-and-deploy-
      spec:
        pipelineRef:
          name: build-and-deploy
        params:
          - { name: git-url, value: $(tt.params.git-url) }
          - { name: git-revision, value: $(tt.params.git-revision) }
        workspaces:
          - name: source
            volumeClaimTemplate:
              spec:
                accessModes: [ReadWriteOnce]
                resources:
                  requests:
                    storage: 1Gi
---
apiVersion: triggers.tekton.dev/v1beta1
kind: EventListener
metadata:
  name: github-listener
spec:
  serviceAccountName: pipeline
  triggers:
    - name: github-push
      interceptors:
        - ref:
            name: github
          params:
            - name: eventTypes
              value: ["push"]
      bindings:
        - ref: github-push
      template:
        ref: build-and-deploy
```

- `TriggerBinding` copies fields from the webhook's JSON body, here the clone URL and the pushed commit, into parameters.
- `TriggerTemplate` defines what to create for each event. It's the same PipelineRun you started by hand, with the parameters filled in from the binding.
- `EventListener` runs a pod and a Service that receive the webhook. The `github` interceptor passes only `push` events, and the listener uses the `pipeline` ServiceAccount to create the PipelineRun.

13. Create the resources and expose the EventListener outside the cluster. Print its URL.
    - **Reference:** [Deployments, References](../../../openshift/deployments/index.md#references) shows `oc apply -f`. [Services, References](../../../openshift/services-networking/services.md#references) shows `oc expose service`. The EventListener's Service is named `el-github-listener`. [Routes, Commands](../../../openshift/services-networking/routes.md#commands) shows how to get a Route's URL. Get the Route `el-github-listener` and print the URL with `echo`.
    - **Check:** you get a URL such as `http://el-github-listener-tekton-lab.apps.example.com`.

14. In your fork, open **Settings > Webhooks > Add webhook**. Set **Payload URL** to the URL above and **Content type** to `application/json`, then save.
    - **Check:** GitHub shows a green check mark for the webhook's first delivery.

    !!! warning "OpenShift Local"
        Your cluster must be reachable from the internet, so this step doesn't work with OpenShift Local.

15. Change the default message in `sample-app/greeting/main.go`, then commit and push.
    - **Reference:** [Tekton Overview](../../../devops/tekton/index.md#tekton-overview) introduces the `tkn` CLI. It doesn't cover listing runs, so use this command: `tkn pipelinerun list`
    - **Check:** a new PipelineRun starts automatically after your push.

!!! tip "Pipelines as Code"
    OpenShift Pipelines also includes **[Pipelines as Code](https://pipelinesascode.com/)**. It keeps PipelineRun definitions in a `.tekton/` directory of your repository, runs them on pushes and pull requests, and reports the results back to GitHub or GitLab. It's the recommended way to run CI on OpenShift.

## Cleanup

Delete the `tekton-lab` project.

- **Reference:** [Projects/Namespaces, References](../../../openshift/core-concepts/namespaces-projects/index.md#references) shows project management. It doesn't cover deleting a project, so use this command:

```bash
oc delete project tekton-lab
```

## What you learned

- [x] Tekton building blocks: Steps, Tasks, Pipelines, runs, workspaces and results
- [x] Reusing the Tasks that ship with OpenShift Pipelines through the cluster resolver
- [x] Building and pushing images with buildah, and deploying by digest
- [x] Triggering pipelines from Git events

## Additional resources

- [Red Hat OpenShift Pipelines documentation](https://docs.redhat.com/en/documentation/red_hat_openshift_pipelines/)
- [Tekton documentation](https://tekton.dev/docs/)
- [Tekton Tasks on Artifact Hub](https://artifacthub.io/packages/search?kind=7)
- [OpenShift Pipelines tutorial](https://github.com/openshift/pipelines-tutorial)
