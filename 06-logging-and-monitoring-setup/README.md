# 06 — Logging and Monitoring setup (OpenShift 4.21)

This topic is a **Day 2 baseline** for observability:

- Platform monitoring (Prometheus/Alertmanager) configuration touchpoints
- Cluster logging operator installation and core configuration concepts
- Log forwarding to external systems (ties into `../04-log-forwarding-to-siem/`)

This repo intentionally keeps manifests **minimal** and focuses on the operational steps and where configuration lives.

## Prerequisites

- OpenShift 4.21 cluster access as `cluster-admin`
- Storage class available (for persistent logging backends, if used)
- Decide whether you want:
  - platform monitoring only (built-in), or
  - platform monitoring + cluster logging + external log forwarding

## Procedure

### 1. Platform monitoring quick checks

```bash
oc -n openshift-monitoring get pods
oc -n openshift-monitoring get svc
```

### 2. Configure core platform monitoring (ConfigMap-driven)

Core platform monitoring is configured via:

- `openshift-monitoring/cluster-monitoring-config` (ConfigMap)

Example (infra placement) is in `../02-infra-nodes-and-monitoring-placement/cluster-monitoring-config.yaml`.

### 3. Install logging components (operator-managed)

Use the OpenShift Logging documentation for 4.21 to install and configure logging components appropriate for your environment and supportability requirements.

At a minimum, verify the logging namespaces and operator installation:

```bash
oc get ns openshift-logging
oc -n openshift-operators get subscriptions | grep -i logging || true
```

### 4. Configure log forwarding

Once logging/collector is available, configure `ClusterLogForwarder` pipelines to send logs to:

- SIEM / syslog (see `../04-log-forwarding-to-siem/clusterlogforwarder-siem-syslog.yaml`)

### 5. Verify

- Monitoring is healthy:

```bash
oc -n openshift-monitoring get co
oc -n openshift-monitoring get pods
```

- Logging components are healthy (if installed):

```bash
oc -n openshift-logging get pods
oc get clusterlogforwarder -n openshift-logging
```

## Expected output

- Platform monitoring pods are Running and stable
- If logging is installed, collector pods are Running
- `ClusterLogForwarder` applies successfully and external destinations receive logs (as configured)

## References (OpenShift 4.21)

- Monitoring stack configuration (OCP docs): `https://docs.openshift.com/container-platform/4.21/monitoring/configuring-the-monitoring-stack.html`
- Configuring core platform monitoring (Red Hat docs): `https://docs.redhat.com/en/documentation/monitoring_stack_for_red_hat_openshift/4.21/html-single/configuring_core_platform_monitoring/`
- Alerts and notifications (Alertmanager, external integrations): `https://docs.redhat.com/en/documentation/monitoring_stack_for_red_hat_openshift/4.21/html/configuring_core_platform_monitoring/configuring-alerts-and-notifications`
- Logging (install, configure, forward): `https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html-single/logging/logging`

