---
tags:
  - Lab
  - Intermediate
  - Observability
  - Metrics
  - Logging
  - Tracing
  - OpenShift
  - Kubernetes
---

# Observability Lab

<span class="lab-badge">45 min</span> <span class="lab-badge">Intermediate</span>

In this lab you'll implement end-to-end cloud-native observability for the `greeting` microservice across the three core signals: querying **metrics** with PromQL, triggering automated **alerts** with PrometheusRule, analyzing distributed **traces** in Jaeger, and correlating **structured logs** via trace IDs.

This lab uses a **tutorial reference** format. Each step gives you a goal and points to the section of the [Observability](../../../openshift/observability/index.md), [Helm & Kustomize](../../../openshift/packaging/index.md), or [Projects/Namespaces](../../../openshift/core-concepts/namespaces-projects/index.md) documentation that shows the command or query syntax you need. **You build the queries and commands yourself** by adapting the documentation examples to this lab. When the documentation doesn't cover something, the step gives you the command and explains what it does.

!!! tip "Keep the documentation open"
    Open [Observability](../../../openshift/observability/index.md) in a second tab before you start. It provides PromQL templates for the RED method (Rate, Errors, Duration) and explains OTel tracing and Prometheus alerting rules.

## Prerequisites

- An OpenShift cluster with **user workload monitoring** enabled (or a local cluster started with `local-cluster.sh --monitoring`). See [Lab Environments](../../../lab-environments.md).
- `oc`, `helm`, and a local clone of the bootcamp repository with your terminal located in `sample-app`:

```bash
cd sample-app
```

## Part 1: Deploy Jaeger and instrument the Greeting service

1. Create a project named `observability-lab` and deploy Jaeger for distributed tracing.
    - **Reference:** [Projects/Namespaces, References](../../../openshift/core-concepts/namespaces-projects/index.md#references). Deploy Jaeger and expose its UI:

    ```bash
    oc new-project observability-lab
    oc apply -f observability/jaeger.yaml
    oc rollout status deployment/jaeger
    oc expose service jaeger --port=ui
    echo "Jaeger UI: http://$(oc get route jaeger -o jsonpath='{.spec.host}')"
    ```

    - **Check:** `oc get deployment/jaeger` shows 1 available replica, and the Route URL opens the Jaeger web UI.

2. Install the `greeting` Helm chart configured with OpenTelemetry tracing pointing to Jaeger and a Prometheus ServiceMonitor.
    - **Reference:** [Helm & Kustomize, Helm](../../../openshift/packaging/index.md#helm) shows `helm install`. [Observability, Scraping your application](../../../openshift/observability/index.md#scraping-your-application) explains ServiceMonitors.

    ```bash
    helm install greeting ./helm/greeting --wait \
      --set tracing.otlpEndpoint=http://jaeger:4318 \
      --set serviceMonitor.enabled=true
    ```

    - **Check:** `oc logs deployment/greeting | grep tracing` outputs `"msg":"tracing enabled","endpoint":"http://jaeger:4318"`.

## Part 2: Scrape and query Prometheus Metrics

3. Inspect the raw Prometheus metrics endpoint exposed by the application.
    - **Reference:** [Observability, OpenTelemetry](../../../openshift/observability/index.md#opentelemetry) describes application metrics endpoints. Run port forwarding and curl:

    ```bash
    oc port-forward service/greeting 8080:8080 &
    PID=$!
    curl -s "localhost:8080/greeting?name=metrics"
    curl -s localhost:8080/fail
    curl -s localhost:8080/metrics | grep ^greeting_
    kill $PID
    ```

    - **Check:** output displays counters `greeting_requests_total` and histogram buckets `greeting_request_duration_seconds_bucket`.

4. Deploy the load generator to produce realistic traffic patterns (normal, failing, and slow requests).
    - Apply the load generator manifest:

    ```bash
    oc apply -f observability/load.yaml
    ```

    - **Check:** `oc get pod load` shows status `Running`.

5. Query the metrics in OpenShift web console under **Observe > Metrics** (or Prometheus UI at `http://localhost:9090`).
    - **Reference:** [Observability, What to measure](../../../openshift/observability/index.md#what-to-measure) shows PromQL queries for the RED method:
        - Request rate by status code: `sum by (code) (rate(greeting_requests_total[5m]))`
        - Error ratio: `sum(rate(greeting_requests_total{code=~"5.."}[5m])) / sum(rate(greeting_requests_total[5m]))`
        - 95th percentile latency: `histogram_quantile(0.95, sum by (le) (rate(greeting_request_duration_seconds_bucket[5m])))`
    - **Check:** the error ratio calculates to approximately `0.33` (~33% failure rate), and p95 latency shows ~`0.47s`.

## Part 3: Configure Prometheus Alerting Rules

6. Create a PrometheusRule alert that triggers when the service error ratio exceeds 10% for over 1 minute.
    - **Reference:** [Observability, Scraping your application](../../../openshift/observability/index.md#scraping-your-application) shows the `PrometheusRule` schema. Apply the alert definition:

    ```bash
    oc apply -f observability/greeting-alerts.yaml
    ```

    - **Check:** in OpenShift console under **Observe > Alerts** (or `http://localhost:9090/alerts`), `GreetingHighErrorRate` transitions from `Pending` to `Firing` after 1 minute.

## Part 4: Analyze Distributed Traces and Correlate Logs

7. Open the Jaeger UI and search for slow traces from the `greeting` service with Min Duration `250ms`.
    - **Reference:** [Observability, OpenTelemetry](../../../openshift/observability/index.md#opentelemetry) describes span hierarchies.
    - **Check:** opening a trace reveals the `slow dependency` child span consuming ~300ms of the total request duration.

8. Correlate JSON log entries to Jaeger traces using the embedded `trace_id`.
    - **Reference:** [Observability, The three signals](../../../openshift/observability/index.md#the-three-signals) explains log-to-trace correlation. Query the logs:

    ```bash
    oc logs deployment/greeting | grep '"path":"/greeting"' | grep -v '"duration_ms":0' | tail -1
    ```

    - Copy the `trace_id` from the log JSON into the Jaeger search bar.
    - **Check:** Jaeger opens the exact distributed trace corresponding to that log line.

## Cleanup

Delete the load generator, uninstall the Helm release, and delete the project.

- **Reference:** [Helm & Kustomize, Helm](../../../openshift/packaging/index.md#helm) and [Projects/Namespaces](../../../openshift/core-concepts/namespaces-projects/index.md#references).

```bash
oc delete pod load
helm uninstall greeting
oc delete project observability-lab
```

| Part | What it does |
| --- | --- |
| `oc delete pod load` | Stops traffic generation |
| `helm uninstall greeting` | Removes the greeting deployment and ServiceMonitor |
| `oc delete project observability-lab` | Deletes the project and Jaeger resources |

## What you learned

- [x] Collecting and calculating RED metrics (Rate, Errors, Duration) with PromQL
- [x] Writing actionable SLO-based alerts using `PrometheusRule` resources
- [x] Exporting OpenTelemetry spans over OTLP to Jaeger for latency root-cause analysis
- [x] Correlating structured application JSON logs with distributed traces via `trace_id`
