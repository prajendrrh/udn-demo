# Use Case 7: Route Advertisements

Demonstrates **route advertisements** for the OVN-Kubernetes network plugin: advertising default pod network and optional **ClusterUserDefinedNetwork (CUDN)** routes to the provider network via BGP, and importing routes from the provider network. Requires a BGP provider (FRR); see [Route advertisements – OpenShift 4.21](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/advanced_networking/route-advertisements).

## What this does

- **Advertise** pod network subnets and EgressIPs to the provider network so external systems can reach pods (and egress IPs) directly.
- **Import** routes from the provider network into the cluster so you don’t need to manually configure routes on each node (alternative to `routingViaHost: true` with static routes).
- **CUDN**: When using CUDNs, you can advertise selected CUDN subnets (e.g. for VM reachability) by labeling those CUDNs and using the CUDN RouteAdvertisements example.

## Prerequisites

- **Bare metal** (route advertisements with BGP are supported on bare metal).
- BGP correctly configured on your network; misconfiguration can disrupt the cluster network.
- **Enable BGP (FRR) and route advertisements** on the Cluster Network Operator (see below).

## Enable route advertisements (cluster-level)

Enable both the FRR routing provider and OVN-Kubernetes route advertisements:

```bash
oc patch Network.operator.openshift.io cluster --type=merge -p '{
  "spec": {
    "additionalRoutingCapabilities": {
      "providers": ["FRR"]
    },
    "defaultNetwork": {
      "ovnKubernetesConfig": {
        "routeAdvertisements": "Enabled"
      }
    }
  }
}'
```

To disable route advertisements (keep FRR if you use it for other things):

```bash
oc patch network.operator cluster --type=merge -p '{
  "spec": {
    "defaultNetwork": {
      "ovnKubernetesConfig": {
        "routeAdvertisements": "Disabled"
      }
    }
  }
}'
```

## What’s included

- **Namespace** `openshift-frr-k8s` (for FRRConfiguration).
- **FRRConfiguration** `receive-all`: base BGP peer configuration with label `routeAdvertisements: receive-all`. **Replace** `neighbor.address` (e.g. `172.18.0.5`) with your route reflector or BGP peer IP. This CR allows receiving all routes from the peer; OVN-Kubernetes generates additional FRRConfiguration objects to advertise cluster routes.
- **RouteAdvertisements** `default`: advertises the **default cluster network** (PodNetwork + EgressIP) using the selected FRRConfiguration.
- **RouteAdvertisements** `advertise-cudns` (optional): advertises **CUDNs** that have the label `export: "true"`. Uncomment in `kustomization.yaml` to apply. Ensure your CUDNs have that label (e.g. in metadata when creating the CUDN).

## Apply order

1. Edit `frrconfiguration-receive-all.yaml`: set `neighbor.address` to your BGP peer/route reflector IP.
2. Apply the base FRRConfiguration **before** any RouteAdvertisements:
   ```bash
   oc apply -k .
   ```
   If you only want the default network advertised, leave `route-advertisements-cudn-example.yaml` commented out in kustomization.

## Verify

```bash
# RouteAdvertisements status (cluster-scoped)
oc get routeadvertisements

# OVN-Kubernetes generates FRRConfiguration objects per node/network
oc get frrconfiguration -n openshift-frr-k8s
# Expect: receive-all + ovnk-generated-* entries
```

## Relation to BGP (use case 6)

- **Use case 6 (BGP)** configures raw FRRConfiguration for custom BGP (prefixes, neighbors, toAdvertise/toReceive).
- **Use case 7 (Route advertisements)** enables CNO-driven route advertisements: you provide a base FRRConfiguration (BGP peers + toReceive), and OVN-Kubernetes **generates** FRRConfiguration objects that advertise pod/CUDN subnets and EgressIPs. Use case 7 is the recommended way to expose pod and CUDN routes to the provider network.

## References

- [Route advertisements – OpenShift 4.21](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/advanced_networking/route-advertisements)
- [About route advertisements](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/advanced_networking/route-advertisements#about-route-advertisements_route-advertisements)
- [Enabling route advertisements](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/advanced_networking/route-advertisements#enabling-route-advertisements_route-advertisements)
- [Example route advertisements setup](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/advanced_networking/route-advertisements#example-route-advertisements-setup_route-advertisements)
