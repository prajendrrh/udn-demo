# Use Case 5: Multihoming (Primary UDN + Secondary NAD)

Demonstrates **multihoming**: each pod has **two interfaces** — a **primary User-Defined Network** (UDN) and a **secondary network** (NAD). The namespace uses a UDN as its primary network (replacing the default cluster network), and pods attach to an additional L2 segment via a `NetworkAttachmentDefinition`.

| Interface | Source | Subnet |
|-----------|--------|--------|
| Primary | UserDefinedNetwork `udn-primary` | 100.5.0.0/24 |
| Secondary | NetworkAttachmentDefinition `l2-secondary` | 10.100.200.0/24 |

## What’s included

- **Namespace** `multihoming-demo` with the primary-UDN label.
- **UserDefinedNetwork** `udn-primary`: primary subnet 100.5.0.0/24.
- **NetworkAttachmentDefinition** `l2-secondary`: secondary subnet 10.100.200.0/24.
- **Deployments** `app-multihomed` (2 replicas) and `test-helper` (1): both use primary UDN + secondary NAD annotation.

## Prerequisites

- OVN-Kubernetes (default in OpenShift). Multihoming (UDN + NAD) and secondary NAD are supported on the same platforms as secondary networks (bare metal, IBM Power, IBM Z, LinuxONE, VMware vSphere, RHOSP).

## Apply

```bash
oc apply -k .
```

Wait for pods:

```bash
oc get pods -n multihoming-demo
```

## Show both IPs (primary UDN + secondary)

Each pod’s `network-status` annotation lists interfaces: primary (UDN 100.5.0.x) and secondary (10.100.200.x).

```bash
echo "=== Pod name | Primary (UDN) | Secondary (NAD) ==="
oc get pods -n multihoming-demo -l app=multihomed -o json | jq -r '.items[] | .metadata.name + " | " + (((.metadata.annotations["k8s.v1.cni.cncf.io/network-status"] // "[]") | fromjson) | map(select(.ips[0] | startswith("100.5.0."))) | .[0].ips[0] // "?") + " | " + (((.metadata.annotations["k8s.v1.cni.cncf.io/network-status"] // "[]") | fromjson) | map(select(.ips[0] | startswith("10.100.200."))) | .[0].ips[0] // "?")'
```

Example: each pod shows one primary IP in 100.5.0.0/24 and one secondary in 10.100.200.0/24.

## Test connectivity (TCP)

Use TCP (no NET_RAW) to verify reachability on both networks. Replace `<PRIMARY_IP>` and `<SECONDARY_IP>` with the other pod’s UDN and NAD IPs from the command above.

1. **On primary (UDN):** From one pod, test TCP to the other pod’s primary IP (e.g. port 80; connection refused is expected if nothing is listening):

   ```bash
   POD1=$(oc get pod -n multihoming-demo -l app=multihomed -o jsonpath='{.items[0].metadata.name}')
   PRIMARY2=$(oc get pods -n multihoming-demo -l app=multihomed -o json | jq -r '.items[1] | (.metadata.annotations["k8s.v1.cni.cncf.io/network-status"] // "[]") | fromjson | map(select(.ips[0] | startswith("100.5.0."))) | .[0].ips[0]')
   oc exec -n multihoming-demo $POD1 -- timeout 2 bash -c "echo >/dev/tcp/$PRIMARY2/80" 2>/dev/null || true
   ```
   Expected: connection refused or timeout; path is reachable if you get “Connection refused” quickly.

2. **On secondary (NAD):** Same idea using the other pod’s secondary IP:

   ```bash
   SECONDARY2=$(oc get pods -n multihoming-demo -l app=multihomed -o json | jq -r '.items[1] | (.metadata.annotations["k8s.v1.cni.cncf.io/network-status"] // "[]") | fromjson | map(select(.ips[0] | startswith("10.100.200."))) | .[0].ips[0]')
   oc exec -n multihoming-demo $POD1 -- timeout 2 bash -c "echo >/dev/tcp/$SECONDARY2/80" 2>/dev/null || true
   ```
   Expected: connection refused or timeout; path is reachable if you get “Connection refused” quickly.

## MultiNetworkPolicy (optional)

The example policy applies to the **secondary** network. To use it:

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
