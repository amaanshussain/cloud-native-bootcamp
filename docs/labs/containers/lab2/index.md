---
tags:
  - Lab
  - Intermediate
  - Containers
  - Registries
  - OpenShift
---

# Image Registry Lab

<span class="lab-badge">40 min</span> <span class="lab-badge">Intermediate</span>

In the [Containers Lab](../lab1/index.md) you pushed the `greeting` image to Quay.io. In this lab you'll run that image on OpenShift, pull it from a **private** repository with a pull secret, and then build the image **inside the cluster** and store it in OpenShift's internal registry.

This lab uses a **tutorial reference** format. Each step gives you a goal and points to the section of the [Container Tool Tutorials](../../../containers/container-tool-tutorials.md), [Registry Tutorials](../../../containers/registry-tutorials.md) or [Image Registries](../../../containers/imageregistry.md) page that covers the idea. **You build the command yourself** when a tutorial shows it. The tutorials teach Podman and Docker, not the OpenShift `oc` CLI, so most steps give you the `oc` command and explain what it does.

!!! tip "Keep the tutorials open"
    Open the [Registry Tutorials](../../../containers/registry-tutorials.md) in a second tab before you start. It uses `my-web-app` where this lab uses `greeting`, and it runs images with Podman or Docker where this lab runs them on OpenShift with `oc`.

## Prerequisites

- You've finished the [Containers Lab](../lab1/index.md), and `quay.io/<your-username>/greeting:v1.0.0` exists and is **public**. If you built it on an Apple Silicon Mac, build it for `linux/amd64` (or multi-arch), as described at the end of that lab.
- You're logged in to an OpenShift cluster with `oc`. See [lab environment options](../../kubernetes/index.md#lab-environment).
- You still have the `greeting` directory with `main.go`, `go.mod` and `Containerfile`.

Set your Quay username and create a project for this lab:

```bash
export QUAY_USER=<your-username>
oc new-project registry-lab
```

| Part | What it does |
| --- | --- |
| `export QUAY_USER=...` | Stores your username in a shell variable, so later commands can use `${QUAY_USER}` |
| `oc new-project registry-lab` | Creates a project (a Kubernetes namespace) and switches to it. Everything in this lab lives in it, so you can delete it in one command at the end |

## Part 1: Deploy an image from a public registry

1. Create a Deployment named `greeting` from your Quay image, with container port `8080`, and wait for the rollout to finish.
    - **Reference:** [Image Registries, image naming](../../../containers/imageregistry.md#image-naming-convention) explains the parts of `quay.io/<your-username>/greeting:v1.0.0`. [Container Tool Tutorials, step 3](../../../containers/container-tool-tutorials.md#3-run-a-container) runs an image on your machine. It doesn't cover running one on OpenShift, so use these commands:

    ```bash
    oc create deployment greeting --image=quay.io/${QUAY_USER}/greeting:v1.0.0 --port=8080
    oc rollout status deployment/greeting
    ```

    | Part | What it does |
    | --- | --- |
    | `oc create deployment greeting` | Creates a Deployment, which keeps a pod running from an image and replaces it when it fails or when the image changes |
    | `--image=...` | The image to run. The full name tells the cluster to pull from Quay.io |
    | `--port=8080` | Records the port the container listens on |
    | `oc rollout status` | Waits until the new pod is running and reports the result |

    ```text title="Expected output"
    deployment.apps/greeting created
    Waiting for deployment "greeting" rollout to finish: 0 of 1 updated replicas are available...
    deployment "greeting" successfully rolled out
    ```

    - **Check:** the last line says `successfully rolled out`.

2. Expose the Deployment as a Service, then expose the Service as a Route.
    - **Reference:** [Container Tool Tutorials, step 3](../../../containers/container-tool-tutorials.md#3-run-a-container) publishes a port with `-p`. On OpenShift, a **Service** gives the pods a stable internal address and a **Route** publishes the Service outside the cluster. The tutorials don't cover either, so use these commands:

    ```bash
    oc expose deployment greeting --port=8080
    oc expose service greeting
    ```

    | Part | What it does |
    | --- | --- |
    | `oc expose deployment greeting --port=8080` | Creates a Service that sends traffic on port `8080` to the Deployment's pods |
    | `oc expose service greeting` | Creates a Route that publishes the Service at a public hostname |

    ```text title="Expected output"
    service/greeting exposed
    route.route.openshift.io/greeting exposed
    ```

    - **Check:** `oc get route greeting` lists a hostname.

3. Call the `/greeting` endpoint through the Route with the name `OpenShift` using `curl`.
    - **Reference:** [Container Tool Tutorials, step 3](../../../containers/container-tool-tutorials.md#3-run-a-container) shows `curl`. Use the Route's hostname instead of `localhost:8080`. The tutorials don't cover reading it, so use this command:

    ```bash
    oc get route greeting -o jsonpath='{.spec.host}'
    ```

    `-o jsonpath='{.spec.host}'` prints only the `host` field of the Route. Put it inside `$( )` to use it in your `curl` URL.

    - **Check:** the response is `{"message":"Welcome to the Cloud Native Bootcamp!","name":"OpenShift"}`.

## Part 2: Pull from a private repository

Most company images live in private repositories. The cluster needs credentials, a **pull secret**, to download them.

4. On [quay.io](https://quay.io/), open the `greeting` repository, go to **Settings**, and make it **private**.
    - **Reference:** [Registry Tutorials](../../../containers/registry-tutorials.md), Red Hat Quay tab, step 4 describes repository visibility.
    - **Check:** the repository shows a **Private** badge.

5. Force a new pull by restarting the Deployment. The new pod fails to start. Find out why.
    - **Reference:** [Container Tool Tutorials, step 4](../../../containers/container-tool-tutorials.md#4-inspect-and-debug-a-running-container) shows how to list, inspect and read the logs of a container. The `oc` equivalents are:

    ```bash
    oc rollout restart deployment/greeting
    oc get pods
    oc describe pod -l app=greeting | grep -A 5 Events
    ```

    | Part | What it does |
    | --- | --- |
    | `oc rollout restart` | Starts a new rollout, so the cluster pulls the image again |
    | `oc get pods` | Lists the pods, like `podman ps` lists containers |
    | `oc describe pod -l app=greeting` | Shows a pod's details and events. `-l app=greeting` selects pods by label |
    | `grep -A 5 Events` | Prints only the `Events` section and the five lines after it |

    ```text title="Expected output"
    NAME                        READY   STATUS             RESTARTS   AGE
    greeting-5d8f7c6b94-x2k7p   1/1     Running            0          6m
    greeting-7c9d6f8b5-q9w4z    0/1     ImagePullBackOff   0          25s

    Events:
      Type     Reason     Age   From               Message
      ----     ------     ----  ----               -------
      Warning  Failed     20s   kubelet            Failed to pull image "quay.io/<your-username>/greeting:v1.0.0": ... unauthorized: access to the requested resource is not authorized
      Warning  Failed     20s   kubelet            Error: ErrImagePull
      Normal   BackOff    8s    kubelet            Back-off pulling image "quay.io/<your-username>/greeting:v1.0.0"
    ```

    - **Check:** the new pod shows `ErrImagePull` or `ImagePullBackOff` and the events say `unauthorized`. The old pod keeps serving traffic, because a rolling update doesn't remove the working pod before its replacement is ready.

6. Create a robot account with read access to the repository:

    1. In Quay, open **Account Settings > Robot Accounts** and create a robot account, for example `openshift_puller`.
    2. Give it **Read** permission on the `greeting` repository.
    3. Open the robot account and copy its username (`<your-username>+openshift_puller`) and token.

    Robot accounts are better than your personal password: each one is scoped to specific repositories and can be revoked on its own. [Image Registries, access control](../../../containers/imageregistry.md#access-control) describes them.

    - **Check:** you have a username ending in `+openshift_puller` and a token.

7. Create a pull secret named `quay-pull` from the robot credentials, and link it to the `default` ServiceAccount for pulling images.
    - **Reference:** [Registry Tutorials](../../../containers/registry-tutorials.md), Red Hat Quay tab, step 1 logs in to Quay from your machine. The cluster needs the same credentials stored as a Secret, which the tutorials don't cover, so use these commands:

    ```bash
    oc create secret docker-registry quay-pull \
      --docker-server=quay.io \
      --docker-username="${QUAY_USER}+openshift_puller" \
      --docker-password='<robot-token>'
    oc secrets link default quay-pull --for=pull
    ```

    | Part | What it does |
    | --- | --- |
    | `oc create secret docker-registry quay-pull` | Creates a Secret named `quay-pull` that holds registry credentials |
    | `--docker-server`, `--docker-username`, `--docker-password` | The registry and the robot account's credentials. Replace `<robot-token>` with your token |
    | `oc secrets link default quay-pull --for=pull` | Adds the Secret to the `default` ServiceAccount's `imagePullSecrets` |

    Every pod that runs as the `default` ServiceAccount can now pull with the Secret. You could also list the Secret under `spec.imagePullSecrets` in each Pod template.

    ```text title="Expected output"
    secret/quay-pull created
    ```

    - **Check:** `oc get secret quay-pull` lists the Secret with type `kubernetes.io/dockerconfigjson`.

8. Delete the failing pod so it's recreated with the pull secret, and check that the rollout completes.
    - **Reference:** [Container Tool Tutorials, step 5](../../../containers/container-tool-tutorials.md#5-stop-and-remove-the-container) removes a container. The Deployment creates a replacement pod when you delete one. The tutorials don't cover selecting pods by status, so use these commands:

    ```bash
    oc delete pod -l app=greeting --field-selector=status.phase=Pending
    oc rollout status deployment/greeting
    ```

    `--field-selector=status.phase=Pending` limits the delete to the pod that is stuck. The working pod stays up.

    - **Check:** the output ends with `deployment "greeting" successfully rolled out`.

## Part 3: Build inside the cluster

OpenShift can build images itself, without Podman on your laptop and without an external registry. A **BuildConfig** describes the build. The result goes into the cluster's **internal registry**, where an **ImageStream** tracks it.

9. From inside your `greeting` directory, create a binary build named `greeting-internal` that uses your Containerfile.
    - **Reference:** [Container Tool Tutorials, step 7](../../../containers/container-tool-tutorials.md#7-build-the-image) shows how to build an image on your machine. The tutorials don't cover building on OpenShift, so use this command:

    ```bash
    cd greeting
    oc new-build --name=greeting-internal --binary --strategy=docker
    ```

    | Part | What it does |
    | --- | --- |
    | `--name=greeting-internal` | Names the BuildConfig and the ImageStream it creates |
    | `--binary` | The source is uploaded from your machine when the build starts, not cloned from Git |
    | `--strategy=docker` | Builds with the Containerfile in the source, like `podman build` |

    ```text title="Expected output"
    --> Creating resources with label build=greeting-internal ...
        imagestream.image.openshift.io "greeting-internal" created
        buildconfig.build.openshift.io "greeting-internal" created
    --> Success
    ```

    - **Check:** the output reports a created `buildconfig` and `imagestream`.

10. Start the build, upload the current directory, and follow the build log.
    - **Reference:** [Container Tool Tutorials, step 7](../../../containers/container-tool-tutorials.md#7-build-the-image) explains the build context, the files sent to the build. The tutorials don't cover starting an OpenShift build, so use this command:

    ```bash
    oc start-build greeting-internal --from-dir=. --follow
    ```

    | Part | What it does |
    | --- | --- |
    | `oc start-build greeting-internal` | Runs the BuildConfig |
    | `--from-dir=.` | Uploads the current directory as the build context |
    | `--follow` | Streams the build log until the build ends |

    - **Check:** the log ends with `Push successful`. The image was pushed to `image-registry.openshift-image-registry.svc:5000/registry-lab/greeting-internal`.

11. Look at the ImageStream that tracks the image.
    - **Reference:** [Container Tool Tutorials, step 2](../../../containers/container-tool-tutorials.md#2-pull-an-image) lists images on your machine. An **ImageStream** is the cluster's equivalent, which the tutorials don't cover, so use these commands:

    ```bash
    oc get imagestream greeting-internal
    oc describe imagestream greeting-internal
    ```

    `oc get` lists the ImageStream and its tags. `oc describe` also shows the image digest behind each tag. [Image Registries, tags vs digests](../../../containers/imageregistry.md#tags-vs-digests) explains the difference.

    ```text title="Expected output"
    NAME                IMAGE REPOSITORY                                                                         TAGS     UPDATED
    greeting-internal   image-registry.openshift-image-registry.svc:5000/registry-lab/greeting-internal   latest   2 minutes ago
    ```

    - **Check:** the ImageStream has a `latest` tag.

12. Deploy the image from the ImageStream with a custom greeting, expose it, and call its `/greeting` endpoint through the Route.
    - **Reference:** [Container Tool Tutorials, step 3](../../../containers/container-tool-tutorials.md#3-run-a-container) runs an image and shows `curl`. The [Containers Lab](../lab1/index.md) set `GREETING` with `-e`. The tutorials don't cover deploying from an ImageStream, so use these commands:

    ```bash
    oc new-app greeting-internal --name=greeting-internal -e GREETING="Built on OpenShift"
    oc expose service greeting-internal
    curl "http://$(oc get route greeting-internal -o jsonpath='{.spec.host}')/greeting"
    ```

    | Part | What it does |
    | --- | --- |
    | `oc new-app greeting-internal` | Creates a Deployment and a Service that run the ImageStream's image |
    | `-e GREETING=...` | Sets the environment variable the app reads, like `-e` in `podman run` |
    | `oc expose service greeting-internal` | Creates a Route for the new Service |

    Because the Deployment uses an ImageStream tag, a later build automatically rolls out the new image.

    ```text title="Expected output"
    {"message":"Built on OpenShift","name":"World"}
    ```

    - **Check:** the `message` in the response is `Built on OpenShift`.

!!! tip "Builds from Git"
    Instead of uploading a local directory, you can build straight from a Git repository: `oc new-build https://github.com/<you>/<repo>.git --strategy=docker`. In the [Tekton lab](../../devops/tekton/index.md) you'll automate build and deploy with a pipeline.

## Cleanup

Remove everything this lab created. The tutorials remove containers and images on your machine, not OpenShift resources, so use this command:

```bash
oc delete project registry-lab
```

Deleting the project deletes its Deployments, Services, Routes, Secrets, BuildConfig and ImageStream. Keep the `greeting` image on Quay and the robot account if you want to repeat the lab.

## What you learned

- [x] Deploying and exposing an image from a public registry
- [x] Pulling from private registries with robot accounts and pull secrets
- [x] Building images in-cluster with BuildConfigs, ImageStreams and the internal registry
