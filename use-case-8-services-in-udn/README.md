# Services: two UDNs (BLUE and RED)

This use case demonstrates **service isolation between two UDNs**. Namespace **blue** is on UDN BLUE, namespace **red** on UDN RED (no default network in this demo). Each has a Service and pods; only pods on the same UDN can reach that UDN’s service—BLUE cannot reach Service red, RED cannot reach Service blue. (For **default network vs one UDN**, see `use-case-udn-services/`.)

| Namespace | Network (UDN) | Subnet       | Service  |
|-----------|---------------|--------------|----------|
| blue      | BLUE          | 100.8.0.0/24 | blue     |
| red       | RED           | 200.3.0.0/24 | red      |

- **Only pods in BLUE** can access Service **blue** (e.g. `http://blue:9376` or `http://blue.blue.svc.cluster.local:9376`).
- **Only pods in RED** can access Service **red** (e.g. `http://red:9376` or `http://red.red.svc.cluster.local:9376`).
- A pod in BLUE cannot reach Service red; a pod in RED cannot reach Service blue (different UDN = isolated).

## What’s included

- **Namespaces** `blue` and `red`, each with the primary-UDN label.
- **UserDefinedNetwork** `udn-blue` (namespace blue), subnet 100.8.0.0/24.
- **UserDefinedNetwork** `udn-red` (namespace red), subnet 200.3.0.0/24.
- **Deployments** `server` in each namespace (agnhost serve-hostname on port 9376).
- **Services** `blue` and `red` (ClusterIP, port 9376) targeting the respective deployment.

## Prerequisites

- OpenShift 4.18+ with UDN support. No extra operators. Project-admin is enough (namespace-scoped UDN).

## Apply

Use **`oc apply -k`** (Kustomize) so resources are applied in the right order (namespaces first, then UDNs, then deployments and services). Do **not** use `oc apply -f` on the directory.

```bash
oc apply -k .
```

Wait for pods:

```bash
oc get pods -n blue
oc get pods -n red
```

## Show UDN IPs

```bash
echo "=== BLUE (100.8.0.0/24) ==="
oc get pods -n blue -o json | jq -r '.items[] | .metadata.name + ": " + ((.metadata.annotations["k8s.v1.cni.cncf.io/network-status"] // "[]") | fromjson | map(select(.ips[0] | startswith("100.8.0."))) | .[0].ips[0] // "?")'

echo "=== RED (200.3.0.0/24) ==="
oc get pods -n red -o json | jq -r '.items[] | .metadata.name + ": " + ((.metadata.annotations["k8s.v1.cni.cncf.io/network-status"] // "[]") | fromjson | map(select(.ips[0] | startswith("200.3.0."))) | .[0].ips[0] // "?")'
```

## Test service access (only same network can reach the service)

Pods use the agnhost image, which includes `curl`. Use a short timeout to show cross-network failure.

1. **From a BLUE pod: access Service blue (same network) — should succeed**

   ```bash
   BLUE_POD=$(oc get pod -n blue -l app=blue -o jsonpath='{.items[0].metadata.name}')
   oc exec -n blue $BLUE_POD -- curl -s -m 3 http://blue:9376
   ```
   Expected: a hostname string (e.g. `server-xxxxx`). So **only pods on BLUE can access Service blue**.

2. **From a BLUE pod: try to access Service red (different network) — should fail**

   ```bash
   oc exec -n blue $BLUE_POD -- curl -s -m 3 http://red.red.svc.cluster.local:9376 || echo "Expected: connection failed (RED not reachable from BLUE)"
   ```
   Expected: connection timeout or failure. **BLUE cannot access Service red.**

3. **From a RED pod: access Service red (same network) — should succeed**

   ```bash
   RED_POD=$(oc get pod -n red -l app=red -o jsonpath='{.items[0].metadata.name}')
   oc exec -n red $RED_POD -- curl -s -m 3 http://red:9376
   ```
   Expected: a hostname string. **Only pods on RED can access Service red.**

4. **From a RED pod: try to access Service blue (different network) — should fail**

   ```bash
   oc exec -n red $RED_POD -- curl -s -m 3 http://blue.blue.svc.cluster.local:9376 || echo "Expected: connection failed (BLUE not reachable from RED)"
   ```
   Expected: connection timeout or failure. **RED cannot access Service blue.**

## Cleanup

```bash
oc delete -k .
```

## References

- [Primary networks (UDN) – OpenShift 4.21](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/multiple_networks/primary-networks)
