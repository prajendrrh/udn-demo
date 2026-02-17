# Use Case: BGP Integration

Bastion **192.168.29.10** as route reflector; cluster nodes (FRR-K8s) peer with it. UDN **extranet** is **10.10.10.0/24**; RouteAdvertisements advertises it so the bastion can reach pods. Edit neighbor IPs in `bastion-frr-config.conf` to match your nodes: `oc get pods -n openshift-frr-k8s -o wide`.

---

## Step 1: Configure FRR on the VM

```bash
sudo dnf install -y frr
sudo systemctl enable frr
```

Copy `bastion-frr-config.conf` to `/etc/frr/frr.conf`, edit neighbor IPs (192.168.29.154, .82, .27) to your FRR-K8s node IPs, then:

```bash
sudo systemctl restart frr
```

If using firewall: `sudo firewall-cmd --permanent --add-port=179/tcp && sudo firewall-cmd --reload`

**Check on bastion:** `sudo vtysh` → `show ip bgp summary` (neighbors Established), `show ip bgp` (routes).

---

## Step 2: Enable FRR and Route Advertisements in OpenShift

```bash
oc patch network.operator cluster --type merge --patch '{
  "spec": {
    "additionalRoutingCapabilities": { "providers": ["FRR"] },
    "defaultNetwork": { "ovnKubernetesConfig": { "routeAdvertisements": "Enabled" } }
  }
}'
```

Verify: `oc get pods -n openshift-frr-k8s -o wide`

---

## Step 3: Apply UDN, FRRConfiguration, RouteAdvertisements, and Pods

Edit `frrconfiguration-bastion.yaml` if needed (bastion IP 192.168.29.10, toReceive 0.0.0.0/0). Then:

```bash
oc apply -k use-case-bgp-integration/
```

Creates: namespace **extranet**, CUDN **extranet** (10.10.10.0/24), FRRConfiguration **receive-filtered**, RouteAdvertisements **default**, deployment **udn-pod**.

**Verify:** `oc get ra`, `oc get cudn`, `oc get frrconfiguration -n openshift-frr-k8s`

---

## Step 4: Test (Bastion → UDN pod)

Get the UDN pod IP (10.10.10.x):

```bash
oc get pod -n extranet -l app=udn-bastion-pod -o json | jq -r '.items[0].metadata.annotations["k8s.v1.cni.cncf.io/network-status"]' | jq -r '.[] | select(.ips[0] | startswith("10.10.10.")) | .ips[0]'
```

From the **bastion VM**, ping that IP: `ping -c 2 <that-ip>`. The bastion learns 10.10.10.0/24 from the cluster via BGP.

---

## Cleanup

```bash
oc delete routeadvertisements default
oc delete -k use-case-bgp-integration/
# Optional: disable RA and FRR
oc patch network.operator cluster --type=merge -p '{"spec":{"defaultNetwork":{"ovnKubernetesConfig":{"routeAdvertisements":"Disabled"}}}}'
oc patch network.operator cluster --type=merge -p '{"spec":{"additionalRoutingCapabilities":null}}'
```

On bastion: stop FRR or revert config.

---

## References

- [BGP routing – OpenShift 4.21](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/advanced_networking/bgp-routing)
- [Route advertisements – OpenShift 4.21](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/advanced_networking/route-advertisements)
