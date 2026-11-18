---
tags:
  - Lab
  - Beginner
  - Workloads
  - OpenShift
  - Kubernetes
---

# Cron Jobs Lab

<span class="lab-badge">20 min</span> <span class="lab-badge">Beginner</span>

In this lab you'll automate recurring scheduled tasks in OpenShift by creating a CronJob, triggering manual executions from the CronJob template, and viewing historical run logs.

This lab uses a **tutorial reference** format. Each step gives you a goal and points to the section of the [Jobs and CronJobs](../../../openshift/pods/jobs/index.md) or [Projects/Namespaces](../../../openshift/core-concepts/namespaces-projects/index.md) documentation that shows the command or syntax you need. **You build the manifest and commands yourself** by adapting the documentation examples to this lab. When the documentation doesn't cover something, the step gives you the command and explains what it does.

!!! tip "Keep the documentation open"
    Open [Jobs and CronJobs](../../../openshift/pods/jobs/index.md) in a second tab before you start. This lab runs every 2 minutes (`*/2 * * * *`) and retains 3 successful job histories (`successfulJobsHistoryLimit: 3`), where the documentation example runs every minute.

## Prerequisites

- You're logged in to an OpenShift cluster with `oc` (or an upstream Kubernetes cluster with `kubectl`). See [Lab Environments](../../../lab-environments.md).
- A terminal and a text editor.

## Part 1: Create a project

1. Create a new project named `lab8` and switch to it.
    - **Reference:** [Projects/Namespaces, References](../../../openshift/core-concepts/namespaces-projects/index.md#references). Create `lab8` instead of `dev`.
    - **Check:** `oc project` shows `Using project "lab8"`.

## Part 2: Write the CronJob manifest

Create a manifest file named `xwing-cronjob.yaml` with the following requirements:

- CronJob name: `xwing-cronjob`
- Schedule: `*/2 * * * *`
- Successful jobs history limit: `3`
- Container image: `registry.access.redhat.com/ubi9/ubi-minimal`
- Command/args: `/bin/sh -c 'date; echo "X-Wing fleet status: all systems go"'`
- Restart policy: `OnFailure`

2. Write and apply `xwing-cronjob.yaml`.
    - **Reference:** [Jobs and CronJobs, References](../../../openshift/pods/jobs/index.md#references) shows the `batch/v1` `CronJob` schema.
    - **Check:** `oc apply -f xwing-cronjob.yaml` outputs `cronjob.batch/xwing-cronjob created`.

## Part 3: Trigger and inspect CronJob executions

3. Trigger an immediate execution of the CronJob without waiting for the scheduled interval.
    - **Reference:** [Jobs and CronJobs, References](../../../openshift/pods/jobs/index.md#references) shows running a CronJob immediately using `oc create job <job-name> --from=cronjob/<cronjob-name>`.
    - Create a job named `xwing-manual` from `xwing-cronjob`.
    - **Check:** `oc create job xwing-manual --from=cronjob/xwing-cronjob` creates the manual Job.

4. Inspect the output logs of the manual Job execution.
    - **Reference:** [Jobs and CronJobs, References](../../../openshift/pods/jobs/index.md#references) shows reading Job logs with `oc logs job/<job-name>`.
    - **Check:** `oc logs job/xwing-manual` outputs a timestamp and `X-Wing fleet status: all systems go`.

5. Verify scheduled Job executions and status history.
    - **Reference:** [Jobs and CronJobs, References](../../../openshift/pods/jobs/index.md#references) shows `oc get cronjobs` and `oc get jobs`.
    - **Check:** `oc get cronjob xwing-cronjob` displays a valid timestamp under `LAST SCHEDULE`, and `oc get jobs` lists up to 3 completed job runs.

## Cleanup

Delete the `lab8` project and all resources created in this lab.

- **Reference:** [Projects/Namespaces, References](../../../openshift/core-concepts/namespaces-projects/index.md#references) shows project management. To delete the project:

```bash
oc delete project lab8
```

| Part | What it does |
| --- | --- |
| `oc delete project lab8` | Deletes the `lab8` project, CronJob, and associated Jobs |

## What you learned

- [x] Defining automated recurring tasks with Kubernetes `CronJob` resources
- [x] Configuring cron schedules, restart policies, and job history limits
- [x] Triggering out-of-band manual job executions from an existing CronJob template
- [x] Querying Job execution logs and scheduling status
