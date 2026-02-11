# Use Case 6: BGP Routing (FRR-K8s)

Demonstrates **Border Gateway Protocol (BGP) routing** on OpenShift using FRR-K8s and the `FRRConfiguration` custom resource. Useful for advertising cluster or user-defined network prefixes to upstream routers and for receiving routes from the network.

## Prerequisites

- **Bare metal** infrastructure (BGP routing is supported on bare metal; see [Supported platforms](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/advanced_networking/bgp-routing#about-border-gateway-protocol-bgp-routing_bgp-routing)).
- Proper BGP setup on your network provider; misconfiguration can cause cluster network issues.
- If using **MetalLB Operator**, BGP/FRR-K8s is enabled automatically; skip step 1 below.

## Enable BGP routing (cluster-level)

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
- **FRRConfiguration** `bgp-demo`: example with one router (ASN 64512), one eBGP multi-hop neighbor, and a prefix to advertise. **Replace** `address`, `asn`, and `prefixes` with your environment before applying.

## Apply (after enabling BGP)

1. Edit `frrconfiguration-example.yaml`: set your BGP peer `address`, peer `asn`, and local `prefixes`.
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
```

## UDN and BGP

With BGP enabled, you can advertise routes for **User-Defined Networks** (e.g. VM subnets on a Layer2 CUDN) to the provider network so external systems can reach workloads. See [Virtual machine reachability over CUDN](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/multiple_networks/primary-networks#about-user-defined-networks_udn) (route advertisements).

## References

- [BGP routing – OpenShift 4.21](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/advanced_networking/bgp-routing)
- [Enabling BGP routing](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/advanced_networking/bgp-routing#enabling-bgp-routing_bgp-routing)
- [Configuring the FRRConfiguration CRD](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/advanced_networking/bgp-routing#configuring-the-frrconfiguration-crd_bgp-routing)
- [Migrating FRR-K8s resources](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/advanced_networking/bgp-routing#migrating-frr-k8s-resources_bgp-routing) (metallb-system → openshift-frr-k8s)
