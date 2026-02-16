# Use Case 5: Multihoming (Primary UDN + Secondary UDN)

Demonstrates **multihoming**: each pod has **two interfaces** — a **primary User-Defined Network** (UDN) and a **secondary UDN**. The namespace uses one UDN as its primary network (replacing the default cluster network), and pods attach to a second UDN via the `k8s.v1.cni.cncf.io/networks` annotation.

| Interface | Source | Subnet |
|-----------|--------|--------|
| Primary | UserDefinedNetwork `udn-primary` (role: Primary) | 100.5.0.0/24 |
| Secondary | UserDefinedNetwork `udn-secondary` (role: Secondary) | 100.6.0.0/24 |

## What’s included

- **Namespace** `multihoming-demo` with the primary-UDN label.
- **UserDefinedNetwork** `udn-primary`: primary subnet 100.5.0.0/24.
- **UserDefinedNetwork** `udn-secondary`: secondary subnet 100.6.0.0/24 (pods attach via annotation).
- **Deployment** `app-multihomed` (2 replicas): each pod has primary UDN + secondary UDN (annotation).

## Prerequisites

- OVN-Kubernetes (default in OpenShift). Multihoming with primary UDN + secondary UDN is supported where user-defined networks are supported. Secondary UDN (role: Secondary) support may vary by OpenShift version; if the CR is rejected, check release notes for Layer2 Secondary role support.

## Apply

```bash
oc apply -k .
```

Wait for pods:

```bash
oc get pods -n multihoming-demo
```

## Show both IPs (primary UDN + secondary UDN)

Each pod’s `network-status` annotation lists interfaces: primary (100.5.0.x) and secondary (100.6.0.x).

```bash
echo "=== Pod name | Primary (UDN) | Secondary (UDN) ==="
oc get pods -n multihoming-demo -l app=multihomed -o json | jq -r '.items[] | .metadata.name + " | " + (((.metadata.annotations["k8s.v1.cni.cncf.io/network-status"] // "[]") | fromjson) | map(select(.ips[0] | startswith("100.5.0."))) | .[0].ips[0] // "?") + " | " + (((.metadata.annotations["k8s.v1.cni.cncf.io/network-status"] // "[]") | fromjson) | map(select(.ips[0] | startswith("100.6.0."))) | .[0].ips[0] // "?")'
```

Example: each pod shows one primary IP in 100.5.0.0/24 and one secondary in 100.6.0.0/24.

## Test connectivity (TCP)

Use TCP (no NET_RAW) to verify reachability on both networks. The commands print a result line so you see whether the path is reachable.

1. **On primary (UDN):** From one pod, test TCP to the other pod’s primary IP (e.g. port 80; connection refused is expected if nothing is listening):

   ```bash
   POD1=$(oc get pod -n multihoming-demo -l app=multihomed -o jsonpath='{.items[0].metadata.name}')
   PRIMARY2=$(oc get pods -n multihoming-demo -l app=multihomed -o json | jq -r '.items[1] | (.metadata.annotations["k8s.v1.cni.cncf.io/network-status"] // "[]") | fromjson | map(select(.ips[0] | startswith("100.5.0."))) | .[0].ips[0]')
   oc exec -n multihoming-demo $POD1 -- timeout 2 bash -c "echo >/dev/tcp/$PRIMARY2/80 2>&1 && echo 'Primary: connected' || echo 'Primary: connection refused (path OK)'"
   ```
   Expected: `Primary: connection refused (path OK)` (path reachable; nothing on 80).

2. **On secondary (UDN):** Same idea using the other pod’s secondary IP:

   ```bash
   SECONDARY2=$(oc get pods -n multihoming-demo -l app=multihomed -o json | jq -r '.items[1] | (.metadata.annotations["k8s.v1.cni.cncf.io/network-status"] // "[]") | fromjson | map(select(.ips[0] | startswith("100.6.0."))) | .[0].ips[0]')
   oc exec -n multihoming-demo $POD1 -- timeout 2 bash -c "echo >/dev/tcp/$SECONDARY2/80 2>&1 && echo 'Secondary: connected' || echo 'Secondary: connection refused (path OK)'"
   ```
   Expected: `Secondary: connection refused (path OK)` (path reachable; nothing on 80).

## MultiNetworkPolicy (optional)

The example policy applies to the **secondary UDN**. To use it:

1. Enable multi-network policy:  
   `oc patch network.operator.openshift.io cluster --type=merge -p '{"spec":{"useMultiNetworkPolicy":true}}'`
2. Add `multi-network-policy-example.yaml` to `kustomization.yaml` and apply again.

## Cleanup

```bash
oc delete -k .
```

## References

- [Primary networks (UDN) – OpenShift 4.21](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/multiple_networks/primary-networks)
- [Secondary networks – OpenShift 4.21](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/multiple_networks/secondary-networks)
- [Configuring multi-network policy](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/multiple_networks/secondary-networks#configuring-multi-network-policy)
