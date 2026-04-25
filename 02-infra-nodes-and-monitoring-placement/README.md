# 02 — Label infra nodes and move monitoring workloads

This topic shows how to:

- **Label infrastructure nodes** (so workloads can be scheduled there intentionally)
- **Move platform monitoring components** (Prometheus/Alertmanager/etc.) onto infra nodes via the Cluster Monitoring Operator configuration

## Prerequisites

- OpenShift 4.21 cluster access as `cluster-admin`
- At least **3 infra nodes** recommended for HA monitoring (anti-affinity rules require spreading pods)
- Nodes exist (or MachineSets exist) that you want to designate as infra

## Procedure

### 1. Identify the target infra nodes

```bash
oc get nodes -o wide
```

### 2. Label nodes as infra

Choose the nodes, then label them.

```bash
oc label node <node1> node-role.kubernetes.io/infra=
oc label node <node2> node-role.kubernetes.io/infra=
oc label node <node3> node-role.kubernetes.io/infra=
```

Verify:

```bash
oc get nodes -L node-role.kubernetes.io/infra
```

### 3. (Optional but common) Taint infra nodes

Taint infra nodes so only workloads with tolerations run there.

```bash
oc adm taint nodes -l node-role.kubernetes.io/infra=node \
  node-role.kubernetes.io/infra=:NoSchedule
```

### 4. Configure platform monitoring to run on infra nodes

1. Review and update `cluster-monitoring-config.yaml` (node selector + tolerations).
2. Apply it:

```bash
oc apply -f ./cluster-monitoring-config.yaml
```

### 5. Verify monitoring pods moved to infra nodes

```bash
oc -n openshift-monitoring get pods -o wide
```

You should see monitoring pods scheduled onto the labeled infra nodes.

## Expected output

- `cluster-monitoring-config` exists in `openshift-monitoring`
- Monitoring components reschedule and stabilize on infra nodes
- `oc -n openshift-monitoring get pods -o wide` shows infra node placement

## References (OpenShift 4.21)

- Monitoring stack configuration (OCP docs): `https://docs.openshift.com/container-platform/4.21/monitoring/configuring-the-monitoring-stack.html`
- Configuring core platform monitoring (Red Hat docs): `https://docs.redhat.com/en/documentation/monitoring_stack_for_red_hat_openshift/4.21/html-single/configuring_core_platform_monitoring/`
- Cluster Monitoring Operator ConfigMap reference: `https://docs.redhat.com/en/documentation/monitoring_stack_for_red_hat_openshift/4.21/html/config_map_reference_for_the_cluster_monitoring_operator/config-map-reference-for-the-cluster-monitoring-operator`

