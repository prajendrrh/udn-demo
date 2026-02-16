# Use Case 6: BGP Routing (FRR-K8s + FRR on bastion VM)

Demonstrates **BGP routing** in two places: (1) **FRR on a bastion VM** (192.168.29.10) as external router and (2) **FRR-K8s on OpenShift**. Example: cluster machine network **192.168.29.0/24**, bastion **192.168.29.10** (advertises 192.168.20.0/24, accepts routes from cluster). Use case 7 adds a UDN 192.168.20.0/24 and route advertisements so a pod can ping 192.168.20.1.

## Prerequisites

- **Bare metal** infrastructure (BGP routing is supported on bare metal; see [Supported platforms](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/advanced_networking/bgp-routing#about-border-gateway-protocol-bgp-routing_bgp-routing)).
- Proper BGP setup on your network provider; misconfiguration can cause cluster network issues.
- If using **MetalLB Operator**, BGP/FRR-K8s is enabled automatically; skip step 1 below.

## 1. Configure FRR on the bastion VM (192.168.29.10)

On the bastion: install FRR (e.g. `dnf install frr`), copy `bastion-frr-config.conf`, replace **CLUSTER_NODE_IP** with one node IP from 192.168.29.0/24 (e.g. 192.168.29.2), then load the config (e.g. `/etc/frr/frr.conf` and restart FRR). The sample advertises **192.168.20.0/24** to the cluster and accepts all routes from the cluster.

## 2. Enable FRR-K8s on OpenShift (cluster-level)

Enable the FRR dynamic routing provider so the FRR-K8s daemon is deployed on nodes:

```bash
oc patch Network.operator.openshift.io/cluster --type=merge -p '{
  "spec": {
    "additionalRoutingCapabilities": {
      "providers": ["FRR"]
    }
  }
}'
```

To disable later:

```bash
oc patch Network.operator.openshift.io/cluster --type=merge -p '{
  "spec": { "additionalRoutingCapabilities": null }
}'
```

## What’s included

- **Namespace** `openshift-frr-k8s` (required for FRRConfiguration in 4.18+).
- **FRRConfiguration** `bgp-demo`: peers with bastion **192.168.29.10** (ASN 64513), cluster ASN 64512, eBGP multi-hop; **toReceive: all** so the cluster learns 192.168.20.0/24 from the bastion.
- **bastion-frr-config.conf**: sample FRR config for the bastion (replace CLUSTER_NODE_IP with a node IP from 192.168.29.0/24).

## Apply (after enabling BGP and configuring bastion FRR)

1. Optional: edit `frrconfiguration-example.yaml` if your bastion IP or ASN differs (default: 192.168.29.10, ASN 64513).
2. Apply:

```bash
oc apply -k .
```

## Verify

```bash
# FRRConfiguration status
oc get frrconfiguration -n openshift-frr-k8s

# FRR-K8s daemon (after BGP is enabled)
oc get pods -n openshift-frr-k8s

# Test pod for connectivity checks
oc get pods -n bgp-connectivity-demo -o wide
```

## Test steps

1. **Confirm FRR-K8s is running**  
   `oc get pods -n openshift-frr-k8s` should show `frr-k8s-*` pods on nodes.

2. **Reach a route learned via BGP (e.g. 192.168.20.1)**  
   With the bastion advertising 192.168.20.0/24, a pod on the default network should reach 192.168.20.1:
   ```bash
   POD=$(oc get pod -n bgp-connectivity-demo -l app=connectivity-test -o jsonpath='{.items[0].metadata.name}')
   oc exec -n bgp-connectivity-demo $POD -- ping -c 2 192.168.20.1
   ```
   Expected: replies if the route is received from the bastion and installed on the node.

3. **Reach the UDN pod from the bastion (after use case 7)**  
   Use case 7 creates a pod on UDN 192.168.100.0/24; the cluster advertises that subnet to the bastion. The bastion config uses `route-map IMPORT in` so it **accepts routes from the cluster**. From the bastion VM, ping the pod IP (192.168.100.x). See use case 7 README for how to get the pod IP and run the ping from the bastion.

## UDN and BGP

With BGP enabled, you can advertise routes for **User-Defined Networks** (e.g. VM subnets on a Layer2 CUDN) to the provider network so external systems can reach workloads. See [Virtual machine reachability over CUDN](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/multiple_networks/primary-networks#about-user-defined-networks_udn) (route advertisements).

## References

- [BGP routing – OpenShift 4.21](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/advanced_networking/bgp-routing)
- [Enabling BGP routing](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/advanced_networking/bgp-routing#enabling-bgp-routing_bgp-routing)
- [Configuring the FRRConfiguration CRD](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/advanced_networking/bgp-routing#configuring-the-frrconfiguration-crd_bgp-routing)
- [Migrating FRR-K8s resources](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/advanced_networking/bgp-routing#migrating-frr-k8s-resources_bgp-routing) (metallb-system → openshift-frr-k8s)
