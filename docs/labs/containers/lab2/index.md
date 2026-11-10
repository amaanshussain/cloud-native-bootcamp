---
tags:
  - Lab
  - Containers
  - Registries
  - OpenShift
  - Intermediate
---

# Image Registry Lab

<span class="lab-badge">40 min</span> <span class="lab-badge">Intermediate</span>

In the [Containers Lab](../lab1/index.md) you pushed the `greeting` image to Quay.io. In this lab you'll run that image on OpenShift, pull it from a **private** repository with a pull secret, and then build the image **inside the cluster** and store it in OpenShift's internal registry.

The lab doesn't hand you every command. **Each step describes what to do. Work out the command yourself**, then check your answer in the [solution](solution.md).

## Prerequisites

- You've finished the [Containers Lab](../lab1/index.md), and `quay.io/<your-username>/greeting:v1.0.0` exists and is **public**. If you built it on an Apple Silicon Mac, build it for `linux/amd64` (or multi-arch), as described at the end of that lab.
- You're logged in to an OpenShift cluster with `oc`. See [lab environment options](../../kubernetes/index.md#lab-environment).
- You still have the `greeting` directory with `main.go`, `go.mod` and `Containerfile`.

```bash
export QUAY_USER=<your-username>
oc new-project registry-lab
```

## Part 1: Deploy an image from a public registry

1. Create a Deployment named `greeting` from your Quay image, with container port `8080`, and wait for the rollout to finish.

2. Expose the Deployment as a Service, then expose the Service as a Route.

3. Call the `/greeting` endpoint through the Route with the name `OpenShift` with `curl`.

```bash
user@server greeting % [REDACTED]
deployment.apps/greeting created
Waiting for deployment "greeting" rollout to finish: 0 of 1 updated replicas are available...
deployment "greeting" successfully rolled out

user@server greeting % [REDACTED]
service/greeting exposed
route.route.openshift.io/greeting exposed

user@server greeting % [REDACTED]
{"message":"Welcome to the Cloud Native Bootcamp!","name":"OpenShift"}
```

## Part 2: Pull from a private repository

Most company images live in private repositories. The cluster needs credentials, a **pull secret**, to download them.

4. On [quay.io](https://quay.io/), open the `greeting` repository, go to **Settings**, and make it **private**.

5. Force a new pull by restarting the Deployment. The new pod fails to start. Find out why.

6. Create a robot account with read access to the repository:

    1. In Quay, open **Account Settings > Robot Accounts** and create a robot account, for example `openshift_puller`.
    2. Give it **Read** permission on the `greeting` repository.
    3. Open the robot account and copy its username (`<your-username>+openshift_puller`) and token.

    Robot accounts are better than your personal password: each one is scoped to specific repositories and can be revoked on its own.

7. Create a pull secret named `quay-pull` from the robot credentials, and link it to the `default` ServiceAccount for pulling images.

8. Delete the failing pod so it's recreated with the pull secret, and check that the rollout completes.

```bash
user@server greeting % [REDACTED]
deployment.apps/greeting restarted

user@server greeting % [REDACTED]
NAME                        READY   STATUS             RESTARTS   AGE
greeting-5d8f7c6b94-x2k7p   1/1     Running            0          6m
greeting-7c9d6f8b5-q9w4z    0/1     ImagePullBackOff   0          25s

user@server greeting % [REDACTED]
Events:
  Type     Reason     Age   From               Message
  ----     ------     ----  ----               -------
  Warning  Failed     20s   kubelet            Failed to pull image "quay.io/<your-username>/greeting:v1.0.0": ... unauthorized: access to the requested resource is not authorized
  Warning  Failed     20s   kubelet            Error: ErrImagePull
  Normal   BackOff    8s    kubelet            Back-off pulling image "quay.io/<your-username>/greeting:v1.0.0"

user@server greeting % [REDACTED]
secret/quay-pull created

user@server greeting % [REDACTED]
pod "greeting-7c9d6f8b5-q9w4z" deleted

user@server greeting % [REDACTED]
deployment "greeting" successfully rolled out
```

## Part 3: Build inside the cluster

OpenShift can build images itself, without Podman on your laptop and without an external registry. A **BuildConfig** describes the build. The result goes into the cluster's **internal registry**, where an **ImageStream** tracks it.

9. From inside your `greeting` directory, create a binary build named `greeting-internal` that uses your Containerfile.

10. Start the build, upload the current directory, and follow the build log.

11. Look at the ImageStream that tracks the image.

12. Deploy the image from the ImageStream with a custom greeting, and expose it. Then call its `/greeting` endpoint through the Route.

```bash
user@server greeting % [REDACTED]
--> Creating resources with label build=greeting-internal ...
    imagestream.image.openshift.io "greeting-internal" created
    buildconfig.build.openshift.io "greeting-internal" created
--> Success

user@server greeting % [REDACTED]
Uploading directory "." as binary input for the build ...
...
Successfully pushed image-registry.openshift-image-registry.svc:5000/registry-lab/greeting-internal@sha256:...
Push successful

user@server greeting % [REDACTED]
NAME                IMAGE REPOSITORY                                                                         TAGS     UPDATED
greeting-internal   image-registry.openshift-image-registry.svc:5000/registry-lab/greeting-internal   latest   2 minutes ago

user@server greeting % [REDACTED]
route.route.openshift.io/greeting-internal exposed

user@server greeting % [REDACTED]
{"message":"Built on OpenShift","name":"World"}
```

!!! tip "Builds from Git"
    Instead of uploading a local directory, you can build straight from a Git repository: `oc new-build https://github.com/<you>/<repo>.git --strategy=docker`. In the [Tekton lab](../../devops/tekton/index.md) you'll automate build and deploy with a pipeline.

## Cleanup

```bash
oc delete project registry-lab
```

## What you learned

- [x] Deploying and exposing an image from a public registry
- [x] Pulling from private registries with robot accounts and pull secrets
- [x] Building images in-cluster with BuildConfigs, ImageStreams and the internal registry
