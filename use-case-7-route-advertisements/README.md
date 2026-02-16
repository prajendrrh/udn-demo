# Use Case 7: Route Advertisements + UDN (reach pod from bastion)

Demonstrates **route advertisements** so the **bastion (FRR on VM) can reach the pod**: cluster machine network **192.168.29.0/24**, bastion **192.168.29.10** (use case 6). This use case adds:

- **CUDN** `cudn-bastion` with subnet **192.168.21.0/24** (separate from bastion network 192.168.20.0/24 so the bastion has no local route and uses the BGP route to the cluster).
- A **pod** in namespace `udn-bastion-demo` on that UDN (pod gets an IP in **192.168.21.0/24**).
- **Route advertisements** so the cluster **advertises** 192.168.21.0/24 to the bastion and **imports** 192.168.20.0/24 from the bastion.

**Goal:** from the **bastion VM** (or any host that peers with the cluster), **ping the pod IP** (192.168.21.x). The cluster advertises the UDN subnet to the bastion; the bastion accepts it and forwards traffic to the cluster. (Pod → 192.168.20.1 already works because the cluster imports 192.168.20.0/24 from the bastion.)

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
- **ClusterUserDefinedNetwork** `cudn-bastion`: subnet **192.168.21.0/24** (advertised to bastion; no overlap with 192.168.20.0/24), label `export: "true"`.
- **FRRConfiguration** `receive-all`: BGP peer **192.168.29.10** (bastion), ASN 64513, eBGP multi-hop; accepts all routes.
- **RouteAdvertisements** `default` and `advertise-cudns`: advertise default network and CUDNs with `export: "true"` (so bastion learns 192.168.21.0/24).
- **Deployment** `udn-pod` in `udn-bastion-demo`: one pod on UDN 192.168.21.0/24 (reachable from bastion).

## Apply order

1. Complete use case 6 (FRR on bastion 192.168.29.10, FRR-K8s on cluster). Enable route advertisements (see above).
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
# Pod should have an IP in 192.168.21.0/24 (check network-status annotation if not in -o wide)

# FRRConfiguration
oc get frrconfiguration -n openshift-frr-k8s
```

## Test steps

1. **Confirm RouteAdvertisements and CUDN**  
   `oc get routeadvertisements` should show status `Accepted`. `oc get cudn` should show `cudn-bastion`. Pod in `udn-bastion-demo` should be Running and get an IP in **192.168.21.0/24**.

2. **Reach the pod from the bastion VM (main goal)**  
   Get the pod's UDN IP (192.168.21.x), then from the **bastion** (192.168.29.10) ping it. The cluster advertises 192.168.21.0/24 to the bastion, so the bastion should have a BGP route and forward traffic to the cluster:
   ```bash
   # From your laptop (or any host with oc)
   oc get pod -n udn-bastion-demo -l app=udn-bastion-pod -o json | jq -r '.items[0].metadata.annotations["k8s.v1.cni.cncf.io/network-status"]' | jq -r '.[] | select(.ips[0] | startswith("192.168.21.")) | .ips[0]'
   # Use that IP (e.g. 192.168.21.2) on the bastion:
   # ssh bastion
   # ping -c 2 192.168.21.2
   ```
   Expected: from the bastion, ping to the pod IP (192.168.21.x) gets replies. The bastion learned 192.168.21.0/24 from the cluster via BGP and sends traffic to the cluster node.

3. **Ping 192.168.20.1 from the UDN pod (outbound)**  
   The cluster imports 192.168.20.0/24 from the bastion, so the pod can reach the bastion network:
   ```bash
   UDN_POD=$(oc get pod -n udn-bastion-demo -l app=udn-bastion-pod -o jsonpath='{.items[0].metadata.name}')
   oc exec -n udn-bastion-demo $UDN_POD -- ping -c 2 192.168.20.1
   ```
   Expected: replies if 192.168.20.1 is reachable from the bastion and the cluster has the imported route.

## Relation to BGP (use case 6)

- **Use case 6 (BGP)** configures raw FRRConfiguration for custom BGP (prefixes, neighbors, toAdvertise/toReceive).
- **Use case 7 (Route advertisements)** enables CNO-driven route advertisements: you provide a base FRRConfiguration (BGP peers + toReceive), and OVN-Kubernetes **generates** FRRConfiguration objects that advertise pod/CUDN subnets and EgressIPs. Use case 7 is the recommended way to expose pod and CUDN routes to the provider network.

## References

- [Route advertisements – OpenShift 4.21](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/advanced_networking/route-advertisements)
- [About route advertisements](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/advanced_networking/route-advertisements#about-route-advertisements_route-advertisements)
- [Enabling route advertisements](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/advanced_networking/route-advertisements#enabling-route-advertisements_route-advertisements)
- [Example route advertisements setup](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/advanced_networking/route-advertisements#example-route-advertisements-setup_route-advertisements)
