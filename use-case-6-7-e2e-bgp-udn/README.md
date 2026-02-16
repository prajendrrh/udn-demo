# Use Case 6+7: End-to-End BGP + UDN + Route Advertisements

Single end-to-end flow: **configure FRR on a VM**, **enable FRR and route advertisements in OpenShift**, **configure a UDN**, **configure BGP/route advertisements in OpenShift**, and **create pods** to test **reach outside from OpenShift** and **reach a pod from the bastion VM**.

**Goal:**

- From a **pod on the default network**: reach an IP on the bastion network (e.g. **172.20.0.1**).
- From the **bastion VM**: reach a **pod on the UDN** (IP in **10.10.10.0/24**).

**Assumptions:** Bastion at **192.168.29.10**, cluster nodes on **192.168.29.0/24**, bastion advertises **172.20.0.0/24** (external network; pods reach 172.20.0.x). **UDN** (pod network in OpenShift) is **10.10.10.0/24**. ASN **64512**. Edit manifests or bastion config if your environment differs.

---

## Step 1: Configure FRR on the VM

1. **Install FRR** on the bastion (RHEL/CentOS):
   ```bash
   sudo dnf install -y frr
   sudo systemctl enable frr
   ```

2. **Copy the sample config** from this repo: `bastion-frr-config.conf`.

3. **Replace `CLUSTER_NODE_IP`** with one OpenShift node IP on the machine network (e.g. **192.168.29.2**). That node will run FRR-K8s and peer with the bastion.

4. **Load the config** (choose one):
   - Copy the edited file to `/etc/frr/frr.conf` and restart FRR: `sudo systemctl restart frr`
   - Or use `vtysh` to paste the relevant `router bgp` and `route-map` sections.

5. **Open BGP port** if a firewall is enabled:
   ```bash
   sudo firewall-cmd --permanent --add-port=179/tcp
   sudo firewall-cmd --reload
   ```

### Troubleshooting: FRR on the VM

Run these on the **bastion**:

```bash
# Enter FRR CLI
sudo vtysh

# Check BGP summary (neighbor state should be Established)
show ip bgp summary

# Check received routes (should see cluster routes, e.g. 10.10.10.0/24 after Step 4–5)
show ip bgp

# Check advertised networks
show ip bgp network

# View full config
show running-config

# Exit
exit
```

- **Neighbor state not Established:** Check connectivity to `CLUSTER_NODE_IP` (ping, port 179). Ensure the cluster has FRR enabled and FRRConfiguration applied so FRR-K8s is peering with the bastion IP.
- **No routes from cluster:** Ensure OpenShift route advertisements are enabled and RouteAdvertisements CR is applied (Steps 2 and 4).

---

## Step 2: Enable FRR and Route Advertisements in OpenShift

Run on your laptop (or any host with `oc`):

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

This enables the FRR routing provider and OVN-Kubernetes route advertisements. The cluster creates the **openshift-frr-k8s** namespace and deploys FRR-K8s daemonset.

### Troubleshooting: FRR and route advertisements in OpenShift

```bash
# 1. Cluster Network Operator: check for FRR and routeAdvertisements
oc get network.operator cluster -o yaml
# Look under spec.additionalRoutingCapabilities.providers for FRR
# and spec.defaultNetwork.ovnKubernetesConfig.routeAdvertisements (Enabled).

# 2. FRR-K8s pods (one per node that runs BGP)
oc get pods -n openshift-frr-k8s -o wide

# 3. FRR-K8s logs (pick a pod name from above)
oc logs -n openshift-frr-k8s -l app=frr-k8s --tail=100

# 4. RouteAdvertisements status (after Step 4)
oc get routeadvertisements
# Status should be Accepted.
```

- **No pods in openshift-frr-k8s:** Wait a few minutes after the patch, or check CNO/operator logs for errors. Bare-metal is required for BGP.
- **routeAdvertisements not Enabled:** Re-apply the patch; ensure `defaultNetwork.ovnKubernetesConfig.routeAdvertisements` is set to `"Enabled"`.

---

## Step 3: Configure UDN in OpenShift

The UDN is defined by a **namespace** (with the right labels) and a **ClusterUserDefinedNetwork (CUDN)**. In this use case they are applied together in Step 4; here we only describe what they do.

- **Namespace `extranet`:** Labels `k8s.ovn.org/primary-user-defined-network: ""` and `network: extranet`. Pods in this namespace get their primary IP from the UDN.
- **CUDN `extranet`:** Selects namespaces with `network: extranet`, provides subnet **10.10.10.0/24** (Layer2), label **advertise: true** so route advertisements will export this subnet to the bastion.

You will apply them in the next step with the rest of the manifests.

---

## Step 4: Configure FRR and BGP Route Advertisements in OpenShift

Edit if needed (bastion IP or advertised prefix):

- **frrconfiguration-bastion.yaml:** `neighbor.address` (default **192.168.29.10**), `toReceive.allowed.prefixes` (default **172.20.0.0/24**).

Then apply all OpenShift resources (UDN + FRR + Route Advertisements + pods):

```bash
oc apply -k use-case-6-7-e2e-bgp-udn/
```

This creates:

- Namespace **extranet** and **bgp-connectivity-demo**
- CUDN **extranet** (10.10.10.0/24, advertise: true)
- **FRRConfiguration** `receive-filtered` in `openshift-frr-k8s` (peer to bastion, receive 172.20.0.0/24, label `use-for-advertisements: true`)
- **RouteAdvertisements** `default` (advertises default network + CUDNs with `advertise: true`)
- Deployments: **connectivity-test** (default network), **udn-pod** (extranet UDN)

### Troubleshooting: FRRConfiguration and RouteAdvertisements

```bash
# 1. FRRConfiguration status
oc get frrconfiguration -n openshift-frr-k8s -o wide

# 2. RouteAdvertisements status
oc get routeadvertisements
# Expect status Accepted.

# 3. CUDN
oc get cudn

# 4. FRR-K8s logs (BGP session with bastion)
oc logs -n openshift-frr-k8s -l app=frr-k8s --tail=200
# Look for BGP session up and prefix exchange.

# 5. If RouteAdvertisements not Accepted
oc describe routeadvertisements default
# Check conditions and events.
```

- **FRRConfiguration not applied:** Ensure namespace `openshift-frr-k8s` exists (created when FRR is enabled). Check spelling and CRD.
- **RouteAdvertisements not Accepted:** Ensure at least one FRRConfiguration has label `use-for-advertisements: true` and is in the same namespace expected by the operator.

---

## Step 5: Create a Pod and Test Reachability

Pods are already created by `oc apply -k` in Step 4. Use them to verify.

### 5a. Pod → outside (default-network pod reaches bastion network)

Pod in **bgp-connectivity-demo** is on the **default network** and should have learned **172.20.0.0/24** via BGP (bastion network). Test with TCP (no ping if NET_RAW is dropped):

```bash
POD=$(oc get pod -n bgp-connectivity-demo -l app=connectivity-test -o jsonpath='{.items[0].metadata.name}')
oc exec -n bgp-connectivity-demo $POD -- timeout 2 bash -c "echo >/dev/tcp/172.20.0.1/80" 2>/dev/null && echo "Reachable" || echo "Connection refused or timeout (path OK if refused)"
```

Or if the image has ping and you allow it:

```bash
oc exec -n bgp-connectivity-demo $POD -- ping -c 2 172.20.0.1
```

**Expected:** Connection refused (nothing on port 80) or reachable — path is working if the route is installed.

### 5b. Bastion → pod (reach UDN pod from VM)

1. Get the UDN pod IP (10.10.10.x):

   ```bash
   oc get pod -n extranet -l app=udn-bastion-pod -o json | jq -r '.items[0].metadata.annotations["k8s.v1.cni.cncf.io/network-status"]' | jq -r '.[] | select(.ips[0] | startswith("10.10.10.")) | .ips[0]'
   ```

2. From the **bastion VM**, ping that IP (e.g. **10.10.10.2**):

   ```bash
   ping -c 2 10.10.10.2
   ```

**Expected:** Replies. The bastion learned 10.10.10.0/24 from OpenShift via BGP and forwards traffic to the cluster.

### Troubleshooting: Reachability

| Symptom | Checks |
|--------|--------|
| Pod cannot reach 172.20.0.1 | On bastion: `show ip bgp` — is 172.20.0.0/24 advertised? On OpenShift: `oc get frrconfiguration`; FRR-K8s logs; confirm toReceive includes 172.20.0.0/24. |
| Bastion cannot reach 10.10.10.x | On bastion: `show ip bgp` — do you see 10.10.10.0/24 from cluster? On OpenShift: `oc get routeadvertisements` Accepted; `oc get cudn`; pod has IP in 10.10.10.0/24 (`oc get pod -n extranet -o wide` or network-status annotation). |
| Pod in extranet has no 10.10.10.x IP | CUDN selector must match namespace labels (`network: extranet`). Restart the pod after ensuring CUDN and namespace exist. |
| BGP session not Established | Firewall (port 179); bastion and node must be routable; FRRConfiguration neighbor address must be the bastion IP; ASN must match (64512). |

---

## Cleanup (reverse order)

```bash
# 1. Delete RouteAdvertisements
oc delete routeadvertisements default

# 2. Delete all resources in this use case
oc delete -k use-case-6-7-e2e-bgp-udn/

# 3. Optional: disable route advertisements and FRR
oc patch network.operator cluster --type=merge -p '{"spec":{"defaultNetwork":{"ovnKubernetesConfig":{"routeAdvertisements":"Disabled"}}}}'
oc patch network.operator cluster --type=merge -p '{"spec":{"additionalRoutingCapabilities":null}}'
```

On the bastion, stop FRR or remove the BGP config if you want a full teardown.

---

## Fresh test (after cleanup)

1. Enable FRR and route advertisements again (Step 2).
2. Configure FRR on the VM (Step 1).
3. Apply: `oc apply -k use-case-6-7-e2e-bgp-udn/`
4. Verify: `oc get ra`, `oc get cudn`, `oc get pods -n openshift-frr-k8s -o wide`, `oc get pods -n extranet -o wide`, `oc get pods -n bgp-connectivity-demo -o wide`
5. Run tests in Step 5.

---

## Summary: What’s in this use case

| Item | Purpose |
|------|--------|
| **bastion-frr-config.conf** | Sample FRR config for VM; replace CLUSTER_NODE_IP. |
| **Namespace extranet** | UDN namespace (label network=extranet). |
| **CUDN extranet** | UDN 10.10.10.0/24, advertise: true. |
| **FRRConfiguration receive-filtered** | BGP peer to 192.168.29.10, receive 172.20.0.0/24 (bastion network), use-for-advertisements: true. |
| **RouteAdvertisements default** | Advertise default network + CUDNs with advertise: true (UDN 10.10.10.0/24). |
| **Namespace bgp-connectivity-demo** | For default-network test pod. |
| **Deployment connectivity-test** | Pod on default network → reach 172.20.0.1 (bastion network). |
| **Deployment udn-pod** | Pod on 10.10.10.0/24 → reach from bastion. |

---

## References

- [BGP routing – OpenShift 4.21](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/advanced_networking/bgp-routing)
- [Route advertisements – OpenShift 4.21](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/advanced_networking/route-advertisements)
- [Primary networks (UDN) – OpenShift 4.21](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/multiple_networks/primary-networks)
