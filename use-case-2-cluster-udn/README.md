# Cluster UDN (multi-namespace shared network)

A **ClusterUserDefinedNetwork (CUDN)** is one network shared by multiple namespaces. Pods in `team-platform-dev`, `team-platform-staging`, and `team-platform-prod` all get IPs from `100.1.0.0/24` and can reach each other across namespaces.

Requires **cluster-admin** (to create the CUDN).

## Apply

```bash
oc apply -k .
```

Wait for pods:

```bash
oc get pods -n team-platform-dev
oc get pods -n team-platform-staging
oc get pods -n team-platform-prod
```

## Show CUDN IPs

`oc get pods -o wide` shows the default cluster IP. To list each pod and its **CUDN IP** (100.1.0.x) from the pod annotation (requires `jq`), select the entry whose IP is in the CUDN subnet:

```bash
echo "=== team-platform-dev (CUDN 100.1.0.0/24) ==="
oc get pods -n team-platform-dev -l app=app-cudn -o json | jq -r '.items[] | .metadata.name + ": " + ((.metadata.annotations["k8s.v1.cni.cncf.io/network-status"] // "[]") | fromjson | map(select(.ips[0] | startswith("100.1.0."))) | .[0].ips[0] // "?")'

echo "=== team-platform-staging (CUDN 100.1.0.0/24) ==="
oc get pods -n team-platform-staging -l app=app-cudn -o json | jq -r '.items[] | .metadata.name + ": " + ((.metadata.annotations["k8s.v1.cni.cncf.io/network-status"] // "[]") | fromjson | map(select(.ips[0] | startswith("100.1.0."))) | .[0].ips[0] // "?")'

echo "=== team-platform-prod (CUDN 100.1.0.0/24) ==="
oc get pods -n team-platform-prod -l app=app-cudn -o json | jq -r '.items[] | .metadata.name + ": " + ((.metadata.annotations["k8s.v1.cni.cncf.io/network-status"] // "[]") | fromjson | map(select(.ips[0] | startswith("100.1.0."))) | .[0].ips[0] // "?")'
```

Example output:

```
=== team-platform-dev (CUDN 100.1.0.0/24) ===
app-xxxxx-abc: 100.1.0.2
app-xxxxx-def: 100.1.0.3
=== team-platform-staging (CUDN 100.1.0.0/24) ===
app-yyyyy-ghi: 100.1.0.4
...
```

## Test connectivity

Use CUDN IPs from **Show CUDN IPs**. Checks use TCP (bash `/dev/tcp`), no `ping`.

**1. Within the same namespace (dev → dev)**  
From one dev pod, connect to another dev pod’s CUDN IP. You should see "Connection refused" (reachable).

```bash
POD_DEV=$(oc get pod -n team-platform-dev -l app=app-cudn -o jsonpath='{.items[0].metadata.name}')
IP_DEV=$(oc get pods -n team-platform-dev -l app=app-cudn -o json | jq -r '.items[1].metadata.annotations["k8s.v1.cni.cncf.io/network-status"] | fromjson | map(select(.ips[0] | startswith("100.1.0."))) | .[0].ips[0]')
echo "From $POD_DEV to dev CUDN IP $IP_DEV (same namespace):"
oc exec -n team-platform-dev $POD_DEV -- bash -c "timeout 2 bash -c 'echo >/dev/tcp/'"$IP_DEV"'/80' 2>&1; echo exit: \$?"
```
Expected: `Connection refused` and exit 1.

**2. Across namespaces (dev → staging)**  
From a dev pod, connect to a staging pod’s CUDN IP. You should also see "Connection refused" (reachable), because they share the same CUDN.

```bash
IP_STAGING=$(oc get pods -n team-platform-staging -l app=app-cudn -o json | jq -r '.items[0].metadata.annotations["k8s.v1.cni.cncf.io/network-status"] | fromjson | map(select(.ips[0] | startswith("100.1.0."))) | .[0].ips[0]')
echo "From $POD_DEV to staging CUDN IP $IP_STAGING (different namespace, same CUDN):"
oc exec -n team-platform-dev $POD_DEV -- bash -c "timeout 2 bash -c 'echo >/dev/tcp/'"$IP_STAGING"'/80' 2>&1; echo exit: \$?"
```
Expected: `Connection refused` and exit 1 (cross-namespace connectivity on the shared CUDN).

## Cleanup

```bash
oc delete -k .
```
