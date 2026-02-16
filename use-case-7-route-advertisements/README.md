# Use Case 7: Route Advertisements + UDN 192.168.20.0/24

Demonstrates **route advertisements** and a **UDN on the bastion network**: cluster machine network **192.168.29.0/24**, bastion **192.168.20.10** (use case 6). This use case adds:

- **CUDN** `cudn-bastion` with subnet **192.168.20.0/24** (same as bastion network).
- A **pod** in namespace `udn-bastion-demo` on that UDN (pod gets an IP in 192.168.20.0/24).
- **Route advertisements** enabled so the cluster advertises this UDN to the bastion and **imports routes** from the bastion (so the pod can reach 192.168.20.1).

**Goal:** ping **192.168.20.1** from the pod. The bastion (use case 6) advertises 192.168.20.0/24 to the cluster so the cluster has a route to 192.168.20.1; the cluster advertises the UDN to the bastion so the bastion can reach the pod.

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
- **Namespace** `udn-bastion-demo` (primary-UDN + `udn-bastion: "true"` for CUDN).
- **ClusterUserDefinedNetwork** `cudn-bastion`: subnet **192.168.20.0/24**, label `export: "true"`.
- **FRRConfiguration** `receive-all`: BGP peer **192.168.20.10** (bastion), ASN 64513, eBGP multi-hop; accepts all routes.
- **RouteAdvertisements** `default` and `advertise-cudns`: advertise default network and CUDNs with `export: "true"`.
- **Deployment** `udn-pod` in `udn-bastion-demo`: one pod on UDN 192.168.20.0/24 (for ping 192.168.20.1).

## Apply order

1. Complete use case 6 (FRR on bastion 192.168.20.10, FRR-K8s on cluster). Enable route advertisements (see above).
2. Apply (namespace, CUDN, FRRConfiguration, RouteAdvertisements, deployments):
   ```bash
   oc apply -k .
   ```

## Verify

```bash
# RouteAdvertisements status (cluster-scoped)
oc get routeadvertisements

# CUDN and UDN pod
oc get cudn
oc get pods -n udn-bastion-demo -o wide
# Pod should have an IP in 192.168.20.0/24 (check network-status annotation if not in -o wide)

# FRRConfiguration
oc get frrconfiguration -n openshift-frr-k8s
```

## Test steps

1. **Confirm RouteAdvertisements and CUDN**  
   `oc get routeadvertisements` should show status `Accepted`. `oc get cudn` should show `cudn-bastion`. Pod in `udn-bastion-demo` should be Running and get an IP in 192.168.20.0/24.

2. **Ping 192.168.20.1 from the UDN pod**  
   The pod is on the UDN 192.168.20.0/24; the bastion advertises that prefix to the cluster so the cluster can route to 192.168.20.1:
   ```bash
   UDN_POD=$(oc get pod -n udn-bastion-demo -l app=udn-bastion-pod -o jsonpath='{.items[0].metadata.name}')
   oc exec -n udn-bastion-demo $UDN_POD -- ping -c 2 192.168.20.1
   ```
   Expected: replies if the route is imported from the bastion and connectivity is correct.

3. **From the bastion (or 192.168.20.x): reach the UDN pod**  
   Get the pod's UDN IP from `oc get pod -n udn-bastion-demo -l app=udn-bastion-pod -o jsonpath='{.metadata.annotations.k8s\.v1\.cni\.cncf\.io/network-status}'` (parse for 192.168.20.x). From the bastion or another host that receives the cluster's BGP advertisements, ping that IP. Expected: replies (CUDN subnet is advertised by the cluster).

## Relation to BGP (use case 6)

- **Use case 6 (BGP)** configures raw FRRConfiguration for custom BGP (prefixes, neighbors, toAdvertise/toReceive).
- **Use case 7 (Route advertisements)** enables CNO-driven route advertisements: you provide a base FRRConfiguration (BGP peers + toReceive), and OVN-Kubernetes **generates** FRRConfiguration objects that advertise pod/CUDN subnets and EgressIPs. Use case 7 is the recommended way to expose pod and CUDN routes to the provider network.

## References

- [Route advertisements – OpenShift 4.21](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/advanced_networking/route-advertisements)
- [About route advertisements](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/advanced_networking/route-advertisements#about-route-advertisements_route-advertisements)
- [Enabling route advertisements](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/advanced_networking/route-advertisements#enabling-route-advertisements_route-advertisements)
- [Example route advertisements setup](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/advanced_networking/route-advertisements#example-route-advertisements-setup_route-advertisements)
