---
tags:
  - Solution
  - Containers
---

# Lab 2 Solution - Image Registry Lab

## Part 1: Deploy an image from a public registry

```bash

export QUAY_USER=<your-username>
oc new-project registry-lab
oc create deployment greeting --image=quay.io/${QUAY_USER}/greeting:v1.0.0 --port=8080
oc rollout status deployment/greeting
oc expose deployment greeting --port=8080
oc expose service greeting
curl "http://$(oc get route greeting -o jsonpath='{.spec.host}')/greeting?name=OpenShift"

```

```text title="Expected output"
user@server greeting % oc create deployment greeting --image=quay.io/${QUAY_USER}/greeting:v1.0.0 --port=8080
deployment.apps/greeting created

user@server greeting % oc rollout status deployment/greeting
Waiting for deployment "greeting" rollout to finish: 0 of 1 updated replicas are available...
deployment "greeting" successfully rolled out

user@server greeting % oc expose deployment greeting --port=8080
service/greeting exposed

user@server greeting % oc expose service greeting
route.route.openshift.io/greeting exposed

user@server greeting % curl "http://$(oc get route greeting -o jsonpath='{.spec.host}')/greeting?name=OpenShift"
{"message":"Welcome to the Cloud Native Bootcamp!","name":"OpenShift"}
```

## Part 2: Pull from a private repository

Make the repository **private** on [quay.io](https://quay.io/) (**Settings**), then restart the Deployment and find out why the new pod fails:

```bash

oc rollout restart deployment/greeting
oc get pods
oc describe pod -l app=greeting | grep -A 5 Events

```

```text title="Expected output"
user@server greeting % oc rollout restart deployment/greeting
deployment.apps/greeting restarted

user@server greeting % oc get pods
NAME                        READY   STATUS             RESTARTS   AGE
greeting-5d8f7c6b94-x2k7p   1/1     Running            0          6m
greeting-7c9d6f8b5-q9w4z    0/1     ImagePullBackOff   0          25s

user@server greeting % oc describe pod -l app=greeting | grep -A 5 Events
Events:
  Type     Reason     Age   From               Message
  ----     ------     ----  ----               -------
  Warning  Failed     20s   kubelet            Failed to pull image "quay.io/<your-username>/greeting:v1.0.0": ... unauthorized: access to the requested resource is not authorized
  Warning  Failed     20s   kubelet            Error: ErrImagePull
  Normal   BackOff    8s    kubelet            Back-off pulling image "quay.io/<your-username>/greeting:v1.0.0"
```

The new pod shows `ErrImagePull` / `ImagePullBackOff`, with an `unauthorized` error in the events. The old pod keeps serving traffic, because the rolling update never removes the working pod before its replacement is ready.

Create a robot account in Quay under **Account Settings > Robot Accounts**, for example `openshift_puller`. Give it **Read** permission on the `greeting` repository and copy its username (`<your-username>+openshift_puller`) and token. Then create the pull secret, link it to the `default` ServiceAccount, and recreate the failing pod:

```bash

oc create secret docker-registry quay-pull \
  --docker-server=quay.io \
  --docker-username="${QUAY_USER}+openshift_puller" \
  --docker-password='<robot-token>'
oc secrets link default quay-pull --for=pull
oc delete pod -l app=greeting --field-selector=status.phase=Pending
oc rollout status deployment/greeting

```

```text title="Expected output"
user@server greeting % oc create secret docker-registry quay-pull ...
secret/quay-pull created

user@server greeting % oc delete pod -l app=greeting --field-selector=status.phase=Pending
pod "greeting-7c9d6f8b5-q9w4z" deleted

user@server greeting % oc rollout status deployment/greeting
deployment "greeting" successfully rolled out
```

!!! tip
    Robot accounts are better than your personal password: each one is scoped to specific repositories and can be revoked on its own.

    `oc secrets link` adds the secret to the ServiceAccount's `imagePullSecrets`. Every pod that runs as that ServiceAccount can then pull with it. You could also list the secret under `spec.imagePullSecrets` in each Pod template.

## Part 3: Build inside the cluster

Run the following from inside the `greeting` directory.

```bash

oc new-build --name=greeting-internal --binary --strategy=docker
oc start-build greeting-internal --from-dir=. --follow
oc get imagestream greeting-internal
oc describe imagestream greeting-internal
oc new-app greeting-internal --name=greeting-internal -e GREETING="Built on OpenShift"
oc expose service greeting-internal
curl "http://$(oc get route greeting-internal -o jsonpath='{.spec.host}')/greeting"

```

```text title="Expected output"
user@server greeting % oc new-build --name=greeting-internal --binary --strategy=docker
--> Creating resources with label build=greeting-internal ...
    imagestream.image.openshift.io "greeting-internal" created
    buildconfig.build.openshift.io "greeting-internal" created
--> Success

user@server greeting % oc start-build greeting-internal --from-dir=. --follow
Uploading directory "." as binary input for the build ...
...
Successfully pushed image-registry.openshift-image-registry.svc:5000/registry-lab/greeting-internal@sha256:...
Push successful

user@server greeting % oc get imagestream greeting-internal
NAME                IMAGE REPOSITORY                                                                   TAGS     UPDATED
greeting-internal   image-registry.openshift-image-registry.svc:5000/registry-lab/greeting-internal   latest   2 minutes ago

user@server greeting % oc expose service greeting-internal
route.route.openshift.io/greeting-internal exposed

user@server greeting % curl "http://$(oc get route greeting-internal -o jsonpath='{.spec.host}')/greeting"
{"message":"Built on OpenShift","name":"World"}
```

!!! tip
    `--binary` means the source is uploaded from your machine when the build starts, not cloned from Git.

    `oc new-app` creates a Deployment and a Service. Because the Deployment uses an ImageStream tag, a later build automatically rolls out the new image.
