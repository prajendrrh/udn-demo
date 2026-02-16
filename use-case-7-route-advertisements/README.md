# Use Case 7: Route Advertisements + UDN (reach pod from bastion)

Demonstrates **route advertisements** so the **bastion (FRR on VM) can reach the pod**: cluster machine network **192.168.29.0/24**, bastion **192.168.29.10** (use case 6). This use case adds:

- **CUDN** `extranet` with subnet **10.10.10.0/24** (separate from bastion network so the bastion has no local route and uses the BGP route to the cluster).
- A **pod** in namespace `extranet` on that UDN (pod gets an IP in **10.10.10.0/24**).
- **Route advertisements** so the cluster **advertises** 10.10.10.0/24 to the bastion and **imports** routes from the bastion (e.g. 172.20.0.0/16 per sample).

**Goal:** from the **bastion VM** (or any host that peers with the cluster), **reach the pod IP** (10.10.10.x). The cluster advertises the UDN subnet to the bastion; the bastion accepts it and forwards traffic to the cluster. (Pod → bastion network is possible if the cluster imports that prefix from the bastion.)

## What this does

- **Advertise** pod network subnets (and optionally EgressIPs) to the provider network so external systems can reach pods directly.
- **Import** routes from the provider network into the cluster via BGP (alternative to static routes).
- **CUDN**: CUDNs with label **advertise: true** are included in the single **RouteAdvertisements** `default` (ClusterUserDefinedNetworks selector), so their subnets (e.g. 10.10.10.0/24) are advertised.

## Prerequisites

- **Bare metal** (route advertisements with BGP are supported on bare metal).
- BGP correctly configured on your network; misconfiguration can disrupt the cluster network.
- **Enable BGP (FRR) and route advertisements** on the Cluster Network Operator (see below).

## Enable route advertisements (cluster-level)

Enable both the FRR routing provider and OVN-Kubernetes route advertisements:

```bash
oc patch network.operator cluster --type=merge -p '{
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

- **Namespace** `openshift-frr-k8s` is created automatically by the cluster when you enable FRR (not in this kustomization).
- **Namespace** `extranet` (primary-UDN + `network: extranet` for CUDN selector).
- **ClusterUserDefinedNetwork** `extranet`: subnet **10.10.10.0/24**, label **advertise: true** (included in RouteAdvertisements).
- **FRRConfiguration** `receive-filtered`: label **use-for-advertisements: true**; BGP peer (e.g. **192.168.111.3** in sample), ASN 64512; **toReceive** mode `filtered` with prefix 172.20.0.0/16. Edit address/prefix to match your bastion.
- **RouteAdvertisements** `default` only: advertises DefaultNetwork (PodNetwork) and ClusterUserDefinedNetworks with **advertise: true** (so bastion learns 10.10.10.0/24).
- **Deployment** `udn-pod` in `extranet`: one pod on UDN 10.10.10.0/24 (reachable from bastion).
- **Connectivity test** namespace and deployment (optional) for outbound tests from a pod.

## Apply order

1. Complete use case 6 (FRR on bastion, FRR-K8s on cluster). Enable route advertisements (see above).
2. Apply (namespace, CUDN, FRRConfiguration, RouteAdvertisements, deployments):
   ```bash
   oc apply -k .
   ```

## Verify

```bash
# RouteAdvertisements status (cluster-scoped)
oc get ra
# or: oc get routeadvertisements

# CUDN and UDN pod
oc get cudn
oc get pods -n extranet -o wide
# Pod should have an IP in 10.10.10.0/24 (check network-status annotation if not in -o wide)

# FRRConfiguration
oc get frrconfiguration -n openshift-frr-k8s

# FRR-K8s pods (sample: on 192.168.111.x)
oc get pods -n openshift-frr-k8s -o wide
```

## Test steps

1. **Confirm RouteAdvertisements and CUDN**  
   `oc get ra` should show status **Accepted**. `oc get cudn` should show **extranet**. Pod in `extranet` should be Running and get an IP in **10.10.10.0/24**.

2. **Reach the pod from the bastion VM (main goal)**  
   Get the pod's UDN IP (10.10.10.x), then from the **bastion** ping or TCP-connect to it. The cluster advertises 10.10.10.0/24 to the bastion, so the bastion should have a BGP route and forward traffic to the cluster:
   ```bash
   # From your laptop (or any host with oc)
   oc get pod -n extranet -l app=udn-bastion-pod -o json | jq -r '.items[0].metadata.annotations["k8s.v1.cni.cncf.io/network-status"]' | jq -r '.[] | select(.ips[0] | startswith("10.10.10.")) | .ips[0]'
   # Use that IP (e.g. 10.10.10.2) on the bastion:
   # ssh bastion
   # ping -c 2 10.10.10.2
   ```
   Expected: from the bastion, ping to the pod IP (10.10.10.x) gets replies.

3. **Reach bastion network from the UDN pod (outbound, TCP)**  
   If the cluster imports the bastion’s prefix (e.g. 172.20.0.0/16), the pod can reach that network. Use TCP (no ping/NET_RAW):
   ```bash
   UDN_POD=$(oc get pod -n extranet -l app=udn-bastion-pod -o jsonpath='{.items[0].metadata.name}')
   oc exec -n extranet $UDN_POD -- timeout 2 bash -c "echo >/dev/tcp/172.20.0.1/80" 2>/dev/null && echo "Reachable" || echo "Connection refused or timeout (path OK if refused)"
   ```

---

## Cleanup (reverse of apply)

Use this to remove use case 7 (and optionally use case 6) so you can re-apply from a clean state.

1. **Delete RouteAdvertisements** (cluster-scoped):
   ```bash
   oc delete routeadvertisements default
   ```

2. **Delete use case 7 resources** (CUDN, namespace, deployments):
   ```bash
   oc delete -k use-case-7-route-advertisements/
   ```
   Or delete manually:
   ```bash
   oc delete cudn extranet
   oc delete namespace extranet
   oc delete namespace route-adv-connectivity-demo   # if present
   ```

3. **Optional – remove use case 6** (FRRConfiguration and connectivity demo):
   ```bash
   oc delete -k use-case-6-bgp-routing/
   ```
   Or:
   ```bash
   oc delete frrconfiguration bgp-demo -n openshift-frr-k8s
   oc delete frrconfiguration receive-filtered -n openshift-frr-k8s
   oc delete namespace bgp-connectivity-demo
   ```

4. **Optional – disable route advertisements and FRR** (cluster-level):
   ```bash
   oc patch network.operator cluster --type=merge -p '{"spec":{"defaultNetwork":{"ovnKubernetesConfig":{"routeAdvertisements":"Disabled"}}}}'
   oc patch network.operator cluster --type=merge -p '{"spec":{"additionalRoutingCapabilities":null}}'
   ```

---

## Fresh test (after cleanup)

1. **Enable FRR and route advertisements** (if you disabled them):
   ```bash
   oc patch network.operator cluster --type=merge -p '{
     "spec": {
       "additionalRoutingCapabilities": { "providers": ["FRR"] },
       "defaultNetwork": { "ovnKubernetesConfig": { "routeAdvertisements": "Enabled" } }
     }
   }'
   ```

2. **Wait for FRR-K8s** (optional):
   ```bash
   oc get pods -n openshift-frr-k8s -o wide
   ```

3. **Apply use case 6**, then **use case 7**:
   ```bash
   oc apply -k use-case-6-bgp-routing/
   oc apply -k use-case-7-route-advertisements/
   ```

4. **Verify**:
   ```bash
   oc get ra
   oc get cudn
   oc get pods -n extranet -o wide
   oc get pods -n openshift-frr-k8s -o wide
   ```

5. Run the **Test steps** above (pod IP from extranet, reach from bastion; optional outbound TCP test).

---

## Relation to BGP (use case 6)

- **Use case 6 (BGP)** configures raw FRRConfiguration for custom BGP (prefixes, neighbors, toReceive).
- **Use case 7 (Route advertisements)** uses CNO-driven route advertisements: you provide an FRRConfiguration with label **use-for-advertisements: true**, and OVN-Kubernetes uses it to advertise pod/CUDN subnets. Use case 7 is the recommended way to expose pod and CUDN routes to the provider network.

## References

- [Route advertisements – OpenShift 4.21](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/advanced_networking/route-advertisements)
- [About route advertisements](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/advanced_networking/route-advertisements#about-route-advertisements_route-advertisements)
- [Enabling route advertisements](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/advanced_networking/route-advertisements#enabling-route-advertisements_route-advertisements)
- [Example route advertisements setup](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/advanced_networking/route-advertisements#example-route-advertisements-setup_route-advertisements)
