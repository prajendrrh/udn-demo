# Use Case 6: BGP Routing (FRR-K8s + FRR on bastion VM)

Demonstrates **BGP routing** in two places: (1) **FRR on a bastion VM** (192.168.29.10) as external router and (2) **FRR-K8s on OpenShift**. Example: cluster machine network **192.168.29.0/24**, bastion **192.168.29.10** (advertises a prefix to the cluster, accepts routes from cluster). Use case 7 adds a UDN and route advertisements so a pod can reach the bastion network.

## Prerequisites

- **Bare metal** infrastructure (BGP routing is supported on bare metal; see [Supported platforms](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/advanced_networking/bgp-routing#about-border-gateway-protocol-bgp-routing_bgp-routing)).
- Proper BGP setup on your network provider; misconfiguration can cause cluster network issues.
- If using **MetalLB Operator**, BGP/FRR-K8s is enabled automatically; skip step 1 below.

## 1. Configure FRR on the bastion VM (192.168.29.10)

On the bastion: install FRR (e.g. `dnf install frr`), copy `bastion-frr-config.conf`, replace **CLUSTER_NODE_IP** with one node IP from 192.168.29.0/24 (e.g. 192.168.29.2), then load the config (e.g. `/etc/frr/frr.conf` and restart FRR). The sample advertises **172.20.0.0/24** (bastion network) to the cluster and accepts all routes from the cluster. UDN in OpenShift is separate (e.g. 10.10.10.0/24 in use case 6+7).

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

- **Namespace** `openshift-frr-k8s` is created automatically by the cluster when you enable FRR (not in this kustomization).
- **FRRConfiguration** `bgp-demo`: one router, ASN 64512; neighbor **192.168.111.3** (edit to your BGP peer, e.g. bastion 192.168.29.10); **toReceive** mode `filtered` with prefix 172.20.0.0/16 (edit to match what the peer advertises).
- **bastion-frr-config.conf**: sample FRR config for the bastion (replace CLUSTER_NODE_IP with a node IP from the machine network).

## Apply (after enabling BGP and configuring bastion FRR)

1. Optional: edit `frrconfiguration-example.yaml` if your BGP peer address or prefix differs (sample: 192.168.111.3, prefix 172.20.0.0/16; or use bastion 192.168.29.10 and your advertised prefix).
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

2. **Reach a route learned via BGP**  
   With the peer advertising the prefix you set in FRRConfiguration (e.g. 172.20.0.0/16), a pod on the default network should reach an IP in that prefix (e.g. 172.20.0.1). Edit the IP to match your peer’s advertised network:
   ```bash
   POD=$(oc get pod -n bgp-connectivity-demo -l app=connectivity-test -o jsonpath='{.items[0].metadata.name}')
   oc exec -n bgp-connectivity-demo $POD -- ping -c 2 172.20.0.1
   ```
   Expected: replies if the route is received from the peer and installed on the node.

**Note:** Pinging the UDN pod (10.10.10.x) from the bastion is not part of use case 6. That is done in **use case 7** (route advertisements + CUDN 10.10.10.0/24); see use case 7 README.

---

## Cleanup (reverse of apply)

To remove use case 6 so you can re-apply from a clean state:

```bash
oc delete -k use-case-6-bgp-routing/
```

Or manually:

```bash
oc delete frrconfiguration bgp-demo -n openshift-frr-k8s
oc delete namespace bgp-connectivity-demo
```

To also remove **use case 7** and cluster-level settings, see the **Cleanup** and **Fresh test** sections in [use-case-7-route-advertisements/README.md](use-case-7-route-advertisements/README.md).

---

## Fresh test (after cleanup)

1. Enable FRR on the cluster (see step 2 above) if you disabled it.
2. Configure FRR on the bastion (see step 1).
3. Edit `frrconfiguration-example.yaml` if needed (neighbor address, prefix).
4. Apply: `oc apply -k use-case-6-bgp-routing/`
5. Verify: `oc get frrconfiguration -n openshift-frr-k8s` and `oc get pods -n openshift-frr-k8s -o wide`.
6. Run the test step (reach bastion network 172.20.0.1 or your peer’s advertised prefix from a pod in `bgp-connectivity-demo`).

For a **full fresh test of use case 6 and 7**, follow the **Fresh test** section in use case 7 README.

---

## UDN and BGP

With BGP enabled, you can advertise routes for **User-Defined Networks** (e.g. VM subnets on a Layer2 CUDN) to the provider network so external systems can reach workloads. See [Virtual machine reachability over CUDN](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/multiple_networks/primary-networks#about-user-defined-networks_udn) (route advertisements).

## References

- [BGP routing – OpenShift 4.21](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/advanced_networking/bgp-routing)
- [Enabling BGP routing](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/advanced_networking/bgp-routing#enabling-bgp-routing_bgp-routing)
- [Configuring the FRRConfiguration CRD](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/advanced_networking/bgp-routing#configuring-the-frrconfiguration-crd_bgp-routing)
- [Migrating FRR-K8s resources](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/advanced_networking/bgp-routing#migrating-frr-k8s-resources_bgp-routing) (metallb-system → openshift-frr-k8s)
