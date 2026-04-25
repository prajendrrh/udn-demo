# 04 — Forward logs to a local SIEM system

This topic forwards OpenShift logs to an external SIEM. The most common pattern is:

- Deploy OpenShift Logging / collector
- Configure a `ClusterLogForwarder` pipeline to send logs to an external endpoint (often **syslog**, TCP/TLS)

## Prerequisites

- OpenShift 4.21 cluster access as `cluster-admin`
- SIEM details (protocol + host/port + TLS requirements) — **not needed for this PoC right now**
- OpenShift Logging components installed (see `../06-logging-and-monitoring-setup/` for baseline setup)

## Procedure

### 1. Create a namespace for logging (if not present)

```bash
oc get ns openshift-logging || oc create ns openshift-logging
```

### 2. (Optional) Create a CA bundle ConfigMap (only if you forward with TLS)

If your SIEM uses a private CA:

```bash
oc -n openshift-logging create configmap siem-ca --from-file=ca.crt=./ca.crt
```

### 3. (Optional) Example `ClusterLogForwarder` (do not apply as-is)

This repo includes `clusterlogforwarder-siem-syslog.yaml` as a **syntax example** only.

If/when you have SIEM details, update the destination and TLS settings and then apply:

```bash
oc apply -f ./clusterlogforwarder-siem-syslog.yaml
```

### 4. Verify logs are flowing

- Confirm collector pods are running (names depend on collector type/version):

```bash
oc -n openshift-logging get pods
```

- Validate the SIEM side is receiving messages (vendor-specific).

## Expected output

- `ClusterLogForwarder` object exists and is accepted by the logging operator
- SIEM receives infrastructure/audit/application logs as configured

## References (OpenShift 4.21)

- Logging docs (includes log forwarding concepts and CRDs): `https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html-single/logging/logging`

