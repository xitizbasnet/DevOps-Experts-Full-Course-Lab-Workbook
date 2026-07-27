# Session 9 — Monitoring with Prometheus & Grafana

## Lab 16 — Prometheus Setup, PromQL & Grafana Dashboards

> **🎯 Objective:** Deploy Prometheus and Grafana on Kubernetes using Helm, write PromQL queries, build monitoring dashboards, and configure alerting.

---

## 🧭 Overview

In this lab, you will build a Kubernetes monitoring stack using:

* **Prometheus** — Metrics collection and monitoring
* **Grafana** — Metrics visualization and dashboards
* **Alertmanager** — Alert routing and notification management
* **Node Exporter** — Host-level system metrics
* **PromQL** — Prometheus Query Language
* **ServiceMonitor** — Kubernetes-native service discovery for Prometheus scraping

The lab progresses from basic monitoring to application instrumentation and SLI/SLO dashboards.

---

# 🏗️ Monitoring Architecture

```text
                    Kubernetes Cluster
                           │
             ┌─────────────┼─────────────┐
             │             │             │
             ▼             ▼             ▼
         Node Exporter   Applications   K8s Metrics
             │             │             │
             └─────────────┼─────────────┘
                           │
                           ▼
                      Prometheus
                           │
                    PromQL Queries
                           │
              ┌────────────┴────────────┐
              ▼                         ▼
           Grafana                 Alertmanager
              │                         │
              ▼                         ▼
        Dashboards                  Notifications
                                        │
                                  Email / Slack
```

---

# 📦 Components Deployed

The `kube-prometheus-stack` Helm chart provides:

```text
Prometheus
Grafana
Alertmanager
Node Exporter
Kubernetes monitoring components
```

These components work together to provide metrics collection, visualization, alerting, and Kubernetes observability.

---

# ⚙️ Deploy with Helm

## 1. Add the Prometheus Community Repository

Add the Helm repository:

```bash
helm repo add prometheus-community \
https://prometheus-community.github.io/helm-charts
```

Update Helm repositories:

```bash
helm repo update
```

---

## 2. Install kube-prometheus-stack

Install the monitoring stack:

```bash
helm install prom-stack \
prometheus-community/kube-prometheus-stack \
--namespace monitoring \
--create-namespace \
--set grafana.adminPassword=Admin@123 \
--set prometheus.prometheusSpec.retention=15d
```

The installation includes:

```text
Prometheus
Grafana
Alertmanager
Node Exporter
```

---

## 3. Verify Monitoring Components

Check the pods:

```bash
kubectl get pods -n monitoring
```

All components should eventually reach:

```text
Running
```

For continuous monitoring:

```bash
kubectl get pods -n monitoring -w
```

You can also inspect services:

```bash
kubectl get svc -n monitoring
```

---

# 📊 Access Grafana

Create a local port-forward:

```bash
kubectl port-forward -n monitoring \
svc/prom-stack-grafana 3000:80
```

Open Grafana:

```text
http://localhost:3000
```

Default credentials configured in this lab:

```text
Username: admin
Password: Admin@123
```

> 🔐 **Security:** `Admin@123` is a lab credential. Do not use this password in a production environment. Store production credentials securely and rotate them according to your organization's security policy.

---

# 🔎 Access Prometheus

Create a port-forward:

```bash
kubectl port-forward -n monitoring \
svc/prom-stack-kube-prometheus-prometheus 9090:9090
```

Open:

```text
http://localhost:9090
```

The Prometheus interface allows you to:

* Execute PromQL queries
* Inspect metrics
* Review query results
* Inspect targets
* Troubleshoot scraping
* Validate monitoring data

---

# 🧮 PromQL Queries

PromQL (**Prometheus Query Language**) is used to retrieve and calculate metrics stored in Prometheus.

---

## CPU Usage per Node

```promql
100 - (avg by(node)(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

This calculates approximate CPU utilization per node based on the percentage of time CPUs are not idle.

---

## Memory Usage Percentage

```promql
(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100
```

This calculates the percentage of memory currently being used.

---

## Pod Restarts in the Last Hour

```promql
increase(kube_pod_container_status_restarts_total[1h]) > 0
```

This identifies containers that have restarted during the previous hour.

---

## HTTP Error Rate

```promql
rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m]) * 100
```

This calculates the percentage of HTTP requests returning 5xx status codes.

---

## Disk Usage

```promql
(1 - node_filesystem_avail_bytes / node_filesystem_size_bytes) * 100
```

This calculates filesystem usage as a percentage.

> 💡 **Tip:** In production, filesystem queries are normally filtered to exclude pseudo-filesystems and other irrelevant mounts. This avoids misleading disk-usage panels.

---

# 🟢 Task Set 1 — Guided

| Task     | What to do                                                                                           |
| -------- | ---------------------------------------------------------------------------------------------------- |
| **T1.1** | Deploy `kube-prometheus-stack` via Helm. Access Grafana. Import dashboard ID `1860` (Node Exporter). |
| **T1.2** | In Prometheus UI, run: `up`. Observe all scrape targets that are `UP`.                               |
| **T1.3** | In Grafana, create a panel: metric = `node_memory_MemAvailable_bytes`. Visualize as time-series.     |

---

## T1.1 — Deploy Monitoring Stack and Import Dashboard

Deploy:

```bash
helm repo add prometheus-community \
https://prometheus-community.github.io/helm-charts

helm repo update

helm install prom-stack \
prometheus-community/kube-prometheus-stack \
--namespace monitoring \
--create-namespace \
--set grafana.adminPassword=Admin@123 \
--set prometheus.prometheusSpec.retention=15d
```

Verify:

```bash
kubectl get pods -n monitoring
```

Access Grafana:

```bash
kubectl port-forward -n monitoring \
svc/prom-stack-grafana 3000:80
```

Open:

```text
http://localhost:3000
```

Log in:

```text
Username: admin
Password: Admin@123
```

### Import Node Exporter Dashboard

In Grafana:

```text
Dashboards
   ↓
New
   ↓
Import
```

Enter:

```text
1860
```

Select the appropriate Prometheus data source and import the dashboard.

Explore:

* CPU usage
* Memory usage
* Disk usage
* Network traffic
* System load
* Node availability

---

## T1.2 — Check Prometheus Targets

Access Prometheus:

```bash
kubectl port-forward -n monitoring \
svc/prom-stack-kube-prometheus-prometheus 9090:9090
```

Open:

```text
http://localhost:9090
```

Run:

```promql
up
```

The result shows whether Prometheus scrape targets are available.

A value of:

```text
1
```

generally indicates the target is reachable and being successfully scraped.

A value of:

```text
0
```

indicates the target is currently unavailable.

You can also inspect targets through:

```text
Status → Targets
```

---

## T1.3 — Create a Memory Panel in Grafana

Create a new dashboard:

```text
Dashboards
   ↓
New
   ↓
New Dashboard
   ↓
Add visualization
```

Use:

```promql
node_memory_MemAvailable_bytes
```

Configure the visualization as:

```text
Time series
```

Set an appropriate unit such as:

```text
Bytes
```

Save the dashboard.

> 💡 **Tip:** A more useful production dashboard often converts raw available-memory bytes into percentage utilization.

---

# 🟡 Task Set 2 — Practice

| Task     | What to do                                                                           |
| -------- | ------------------------------------------------------------------------------------ |
| **T2.1** | Write PromQL for pod restart count. Create Grafana panel with red threshold at `>3`. |
| **T2.2** | Configure Alertmanager to send email when CPU `> 80%` for 5 min.                     |
| **T2.3** | Import K8s cluster dashboard (ID `6417`). Understand each panel.                     |

---

# T2.1 — Pod Restart Monitoring

Use:

```promql
increase(kube_pod_container_status_restarts_total[1h])
```

This displays the increase in container restart counters over the previous hour.

To identify containers with more than three restarts:

```promql
increase(kube_pod_container_status_restarts_total[1h]) > 3
```

Create a Grafana panel using the query.

Configure a threshold:

```text
Warning: > 3
```

The dashboard should make abnormal restart activity immediately visible.

---

# T2.2 — CPU Alerting with Alertmanager

The objective is to trigger an alert when CPU utilization remains above 80% for five minutes.

A suitable CPU query is:

```promql
100 - (avg by(node)(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
```

The alert should conceptually behave as:

```text
CPU > 80%
     │
     │
     ▼
Condition remains true
for 5 minutes
     │
     ▼
Prometheus Alert
     │
     ▼
Alertmanager
     │
     ▼
Email notification
```

Configure Alertmanager with:

* SMTP server
* SMTP username
* SMTP password
* Sender address
* Recipient address
* Alert routing rules

> 🔐 **Security:** Do not commit SMTP passwords or other credentials to Git. Use Kubernetes Secrets or an appropriate external secret-management system.

---

# T2.3 — Import Kubernetes Cluster Dashboard

In Grafana:

```text
Dashboards
   ↓
New
   ↓
Import
```

Enter dashboard ID:

```text
6417
```

Select the Prometheus data source.

Review the dashboard panels and identify:

* Cluster CPU utilization
* Memory utilization
* Pod count
* Node health
* Kubernetes workloads
* Resource requests
* Resource limits
* Network information
* Container performance

---

# 🔴 Task Set 3 — Challenge

| Task     | What to do                                                                                                 |
| -------- | ---------------------------------------------------------------------------------------------------------- |
| **T3.1** | Instrument a Flask app with `prometheus_client`. Expose `/metrics`. Add `ServiceMonitor` CRD to scrape it. |
| **T3.2** | Configure Alertmanager with Slack webhook. Test: `kubectl delete pod` — alert fires to Slack.              |
| **T3.3** | Build a Grafana dashboard with SLI/SLO panels: availability %, p95 latency, error rate.                    |

---

# T3.1 — Instrument a Flask Application

Install the Prometheus Python client:

```bash
pip install prometheus_client
```

A Flask application can expose Prometheus metrics through:

```text
/metrics
```

Example application structure:

```text
flask-app/
├── app.py
├── requirements.txt
└── Dockerfile
```

Add the Prometheus client to:

```text
requirements.txt
```

Example:

```text
Flask
prometheus_client
```

The application should expose:

```text
http://<application>:<port>/metrics
```

Prometheus can then scrape this endpoint.

---

# 📡 ServiceMonitor

The `ServiceMonitor` custom resource allows the Prometheus Operator to discover and scrape Kubernetes services.

Conceptually:

```text
Flask Application
       │
       ▼
 /metrics endpoint
       │
       ▼
 Kubernetes Service
       │
       ▼
 ServiceMonitor
       │
       ▼
 Prometheus
       │
       ▼
 Grafana
```

Create a Kubernetes Service for the Flask application and define a `ServiceMonitor` targeting that Service.

Example structure:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: flask-app
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: flask-app
  endpoints:
    - port: http
      path: /metrics
      interval: 15s
```

> ⚠️ **Important:** The ServiceMonitor selector and Service labels must match. Also ensure the Prometheus instance is configured to discover ServiceMonitors from the namespace and labels you use.

---

# T3.2 — Alertmanager Slack Integration

Configure Alertmanager to send notifications to a Slack webhook.

The workflow is:

```text
Kubernetes Pod
       │
       │ Failure
       ▼
Prometheus
       │
       │ Alert condition
       ▼
Alertmanager
       │
       │ Slack receiver
       ▼
Slack Channel
```

Test the monitoring workflow by deleting a pod:

```bash
kubectl delete pod <pod-name> -n <namespace>
```

Monitor:

```bash
kubectl get pods -n <namespace> -w
```

The pod should be recreated if it is managed by a Deployment or another controller.

Verify:

1. Prometheus detects the relevant condition.
2. The alert enters the pending/firing state.
3. Alertmanager receives the alert.
4. Alertmanager routes it to Slack.
5. The notification appears in the configured Slack channel.

> 💡 **Note:** A pod deletion by itself does not automatically guarantee a Prometheus alert. The alert must be based on a condition represented by a metric and defined in the alerting rules.

---

# T3.3 — Build an SLI/SLO Dashboard

Create a Grafana dashboard containing:

```text
┌──────────────────────────────────────────┐
│             SERVICE HEALTH               │
├────────────────┬─────────────────────────┤
│ Availability % │ p95 Latency             │
├────────────────┼─────────────────────────┤
│ Error Rate     │ Request Rate             │
└────────────────┴─────────────────────────┘
```

---

## Availability %

An example availability calculation is:

```promql
sum(rate(http_requests_total{status!~"5.."}[5m]))
/
sum(rate(http_requests_total[5m]))
* 100
```

This estimates the percentage of requests that did not return 5xx responses.

---

## Error Rate

Example:

```promql
sum(rate(http_requests_total{status=~"5.."}[5m]))
/
sum(rate(http_requests_total[5m]))
* 100
```

This shows the percentage of requests returning server errors.

---

## p95 Latency

If the application exports a Prometheus histogram such as:

```text
http_request_duration_seconds_bucket
```

a p95 query can be:

```promql
histogram_quantile(
  0.95,
  sum by (le) (
    rate(http_request_duration_seconds_bucket[5m])
  )
)
```

This estimates the 95th percentile request latency.

---

# 📈 Recommended SLI/SLO Dashboard

Create panels for:

### Availability

```text
Target: ≥ 99.9%
```

### p95 Latency

```text
Target: Define according to application requirements
```

### Error Rate

```text
Target: Keep below the agreed service threshold
```

### Request Rate

```text
Requests per second
```

### Pod Restarts

```promql
increase(kube_pod_container_status_restarts_total[1h])
```

### CPU Usage

```promql
100 - (avg by(node)(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

### Memory Usage

```promql
(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100
```

---

# 🧪 Monitoring Validation Workflow

Use the following workflow to validate the monitoring stack:

```text
1. Deploy Application
        │
        ▼
2. Expose Metrics
        │
        ▼
3. Create Kubernetes Service
        │
        ▼
4. Create ServiceMonitor
        │
        ▼
5. Prometheus Discovers Target
        │
        ▼
6. Verify Target = UP
        │
        ▼
7. Run PromQL Query
        │
        ▼
8. Create Grafana Panel
        │
        ▼
9. Create Alert Rule
        │
        ▼
10. Trigger Test Condition
        │
        ▼
11. Alertmanager Routes Alert
        │
        ▼
12. Verify Notification
```

---

# 🔧 Useful Troubleshooting Commands

## Check Monitoring Pods

```bash
kubectl get pods -n monitoring
```

## Check Monitoring Services

```bash
kubectl get svc -n monitoring
```

## Check Prometheus Targets

```text
Prometheus UI
→ Status
→ Targets
```

## Check ServiceMonitors

```bash
kubectl get servicemonitor -A
```

## Describe a ServiceMonitor

```bash
kubectl describe servicemonitor <name> -n <namespace>
```

## Check Prometheus Rules

```bash
kubectl get prometheusrules -A
```

## Check Alertmanager

```bash
kubectl get pods -n monitoring | grep alertmanager
```

## Check Grafana

```bash
kubectl get pods -n monitoring | grep grafana
```

## Check Application Metrics

```bash
curl http://<service-ip>:<port>/metrics
```

---

# 🩺 Common Monitoring Issues

## Prometheus Target Shows `DOWN`

Check:

* Kubernetes Service
* Service labels
* ServiceMonitor selector
* Service port name
* Metrics endpoint
* Application availability
* Prometheus ServiceMonitor discovery configuration

Useful commands:

```bash
kubectl get svc -A
```

```bash
kubectl get servicemonitor -A
```

```bash
kubectl describe servicemonitor <name> -n <namespace>
```

---

## Grafana Shows No Data

Check:

1. Grafana datasource.
2. Prometheus availability.
3. PromQL query.
4. Selected time range.
5. Prometheus targets.
6. Metric name.
7. Dashboard variables.

Test the query directly in Prometheus first.

---

## Application `/metrics` Does Not Work

Test:

```bash
curl http://<application>:<port>/metrics
```

If running inside Kubernetes, test connectivity from another pod:

```bash
kubectl run debug \
--image=curlimages/curl \
-it --rm -- sh
```

Then:

```bash
curl http://<service-name>:<port>/metrics
```

---

## Alerts Are Not Firing

Check:

* Alert rule expression
* Evaluation interval
* `for` duration
* Prometheus rule configuration
* Prometheus targets
* Alertmanager configuration
* Alert routing
* Notification receiver

Inspect:

```text
Prometheus
→ Alerts
```

And:

```text
Alertmanager
→ Alerts
```

---

# 🛡️ Monitoring & Security Best Practices

* Never use default Grafana passwords in production.
* Store credentials securely.
* Do not commit SMTP credentials to Git.
* Protect Grafana administrative access.
* Restrict Prometheus and Alertmanager network exposure.
* Use TLS for externally exposed monitoring interfaces.
* Use RBAC for Kubernetes monitoring components.
* Avoid exposing Prometheus directly to the public Internet.
* Use appropriate retention periods for metrics.
* Monitor Prometheus storage consumption.
* Define meaningful alert thresholds.
* Avoid excessive alerting that creates alert fatigue.
* Use SLI/SLO-based alerting for critical services.
* Monitor both infrastructure and application-level metrics.
* Document alert ownership and escalation paths.
* Test alerts periodically.
* Review dashboards regularly.
* Use labels consistently across applications and environments.

---

# 📋 Lab 16 Completion Checklist

* [ ] Add Prometheus Community Helm repository.
* [ ] Update Helm repositories.
* [ ] Install `kube-prometheus-stack`.
* [ ] Create the `monitoring` namespace.
* [ ] Verify Prometheus is running.
* [ ] Verify Grafana is running.
* [ ] Verify Alertmanager is running.
* [ ] Verify Node Exporter is running.
* [ ] Access Grafana.
* [ ] Access Prometheus.
* [ ] Import Node Exporter dashboard `1860`.
* [ ] Run the PromQL query `up`.
* [ ] Review Prometheus scrape targets.
* [ ] Create a Grafana memory panel.
* [ ] Create a pod restart monitoring panel.
* [ ] Configure a restart threshold.
* [ ] Configure CPU alerting.
* [ ] Configure email notifications.
* [ ] Import Kubernetes dashboard `6417`.
* [ ] Instrument a Flask application.
* [ ] Expose `/metrics`.
* [ ] Create a Kubernetes Service.
* [ ] Create a `ServiceMonitor`.
* [ ] Verify Prometheus discovers the application.
* [ ] Configure Slack notifications.
* [ ] Test an alert condition.
* [ ] Build an SLI/SLO dashboard.
* [ ] Add availability percentage.
* [ ] Add p95 latency.
* [ ] Add error rate.
* [ ] Add request rate.
* [ ] Review monitoring security best practices.

---

# 🧠 Lab 16 — Key Takeaways

| Component / Concept       | Purpose                                                       |
| ------------------------- | ------------------------------------------------------------- |
| **Prometheus**            | Collects and stores time-series metrics                       |
| **PromQL**                | Queries and calculates Prometheus metrics                     |
| **Grafana**               | Visualizes metrics through dashboards                         |
| **Alertmanager**          | Routes and manages alerts                                     |
| **Node Exporter**         | Provides host-level system metrics                            |
| **kube-prometheus-stack** | Helm-based Kubernetes monitoring stack                        |
| **ServiceMonitor**        | Defines Kubernetes service scraping for Prometheus Operator   |
| **`up` metric**           | Indicates whether a scrape target is available                |
| **SLI**                   | Service Level Indicator used to measure service behavior      |
| **SLO**                   | Service Level Objective defining a desired reliability target |
| **p95 latency**           | Latency value below which 95% of observations fall            |
| **Error rate**            | Percentage of requests resulting in errors                    |
| **Dashboard**             | Visual representation of operational metrics                  |
| **Alert**                 | Notification triggered when a defined condition is met        |

---

# 🚀 End-to-End Observability Workflow

```text
                    APPLICATION
                         │
                         │ /metrics
                         ▼
                  Kubernetes Service
                         │
                         ▼
                    ServiceMonitor
                         │
                         ▼
                     Prometheus
                         │
              ┌──────────┴──────────┐
              │                     │
           PromQL                Alert Rules
              │                     │
              ▼                     ▼
           Grafana              Alertmanager
              │                     │
              │              ┌──────┴──────┐
              │              ▼             ▼
              │            Email         Slack
              │
              ▼
       Dashboards & SLOs
              │
              ▼
      Operational Visibility
              │
              ▼
       Faster Troubleshooting
```

> **🎓 Lab Outcome:** By completing Lab 16, you should be able to deploy the Prometheus/Grafana monitoring stack on Kubernetes, query infrastructure and application metrics using PromQL, build operational dashboards, configure Alertmanager notifications, instrument a Flask application, expose application metrics through `/metrics`, configure a `ServiceMonitor`, and create SLI/SLO-focused observability dashboards.
