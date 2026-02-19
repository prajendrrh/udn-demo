# Overlapping pod IPs

Two namespaces each have their own **UserDefinedNetwork** using the **same subnet** (`100.4.0.0/24`). Pods in `overlapping-a` and pods in `overlapping-b` can get IPs from the same range (e.g. both can have `100.4.0.2`). There is no conflict because each UDN is a separate logical network; overlapping IPs are allowed across UDNs.

## Apply

```bash
oc apply -k .
```

Wait for pods:

```bash
oc get pods -n overlapping-a
oc get pods -n overlapping-b
```

## Show UDN IPs

Both namespaces use the same subnet, so you can see the same IP (e.g. 100.4.0.2) in each (overlapping):

```bash
echo "=== overlapping-a (UDN 100.4.0.0/24) ==="
oc get pods -n overlapping-a -l app=app-overlap -o json | jq -r '.items[] | .metadata.name + ": " + ((.metadata.annotations["k8s.v1.cni.cncf.io/network-status"] // "[]") | fromjson | map(select(.ips[0] | startswith("100.4.0."))) | .[0].ips[0] // "?")'

echo "=== overlapping-b (UDN 100.4.0.0/24, same subnet) ==="
oc get pods -n overlapping-b -l app=app-overlap -o json | jq -r '.items[] | .metadata.name + ": " + ((.metadata.annotations["k8s.v1.cni.cncf.io/network-status"] // "[]") | fromjson | map(select(.ips[0] | startswith("100.4.0."))) | .[0].ips[0] // "?")'
```

Example: both might show `100.4.0.2` — overlapping IPs, different logical networks.

## Test connectivity

- **Within a namespace:** Same-namespace pods on the same UDN can reach each other by UDN IP (e.g. scale to 2 in one namespace and run a TCP test from one pod to the other’s UDN IP).
- **Across namespaces:** Pods in `overlapping-a` and `overlapping-b` are on different UDNs (different logical networks). They are isolated even if their UDN IPs are the same; there is no connectivity between the two namespaces.

## Cleanup

```bash
oc delete -k .
```
