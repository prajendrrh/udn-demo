# Use Case 3: Layer2 vs Layer3 UDN Topologies

Compare **Layer2** (single L2 segment) and **Layer3** (per-node subnets, L3 routing) User-Defined Networks.

- **Layer2** (`udn-layer2-demo`): All pods on one subnet `100.2.0.0/24`. Single logical switch.
- **Layer3** (`udn-layer3-demo`): CIDR `200.1.0.0/24` split into per-node /24s; L3 routing between nodes. Pod IPs are 200.1.x.x.

## Apply

```bash
oc apply -k .
```

Wait for pods:

```bash
oc get pods -n udn-layer2-demo
oc get pods -n udn-layer3-demo
```

## Show UDN IPs

`oc get pods -o wide` shows the default cluster IP. To list each pod and its UDN IP from the annotation (requires `jq`), select by subnet:

```bash
echo "=== udn-layer2-demo (Layer2 100.2.0.0/24) ==="
oc get pods -n udn-layer2-demo -l app=app-layer2 -o json | jq -r '.items[] | .metadata.name + ": " + ((.metadata.annotations["k8s.v1.cni.cncf.io/network-status"] // "[]") | fromjson | map(select(.ips[0] | startswith("100.2.0."))) | .[0].ips[0] // "?")'

echo "=== udn-layer3-demo (Layer3 200.1.x.x) ==="
oc get pods -n udn-layer3-demo -l app=app-layer3 -o json | jq -r '.items[] | .metadata.name + ": " + ((.metadata.annotations["k8s.v1.cni.cncf.io/network-status"] // "[]") | fromjson | map(select(.ips[0] | startswith("200.1."))) | .[0].ips[0] // "?")'
```

Example output:

```
=== udn-layer2-demo (Layer2 100.2.0.0/24) ===
app-xxxxx-abc: 100.2.0.2
app-xxxxx-def: 100.2.0.3
=== udn-layer3-demo (Layer3 200.1.x.x) ===
app-yyyyy-ghi: 200.1.0.2
app-yyyyy-jkl: 200.1.1.3
```

## Test connectivity

Use UDN IPs from **Show UDN IPs**. Checks use TCP (bash `/dev/tcp`), no `ping`.

**1. Layer2: same namespace (pod → pod)**  
From one layer2 pod, connect to another layer2 pod’s UDN IP. You should see "Connection refused" (reachable).

```bash
POD_L2=$(oc get pod -n udn-layer2-demo -l app=app-layer2 -o jsonpath='{.items[0].metadata.name}')
IP_L2=$(oc get pods -n udn-layer2-demo -l app=app-layer2 -o json | jq -r '.items[1].metadata.annotations["k8s.v1.cni.cncf.io/network-status"] | fromjson | map(select(.ips[0] | startswith("100.2.0."))) | .[0].ips[0]')
echo "From $POD_L2 to layer2 UDN IP $IP_L2:"
oc exec -n udn-layer2-demo $POD_L2 -- bash -c "timeout 2 bash -c 'echo >/dev/tcp/'"$IP_L2"'/80' 2>&1; echo exit: \$?"
```
Expected: `Connection refused` and exit 1.

**2. Layer3: same namespace (pod → pod, same or different node)**  
L3 routing allows connectivity across per-node subnets. From one layer3 pod, connect to another’s UDN IP.

```bash
POD_L3=$(oc get pod -n udn-layer3-demo -l app=app-layer3 -o jsonpath='{.items[0].metadata.name}')
IP_L3=$(oc get pods -n udn-layer3-demo -l app=app-layer3 -o json | jq -r '.items[1].metadata.annotations["k8s.v1.cni.cncf.io/network-status"] | fromjson | map(select(.ips[0] | startswith("200.1."))) | .[0].ips[0]')
echo "From $POD_L3 to layer3 UDN IP $IP_L3:"
oc exec -n udn-layer3-demo $POD_L3 -- bash -c "timeout 2 bash -c 'echo >/dev/tcp/'"$IP_L3"'/80' 2>&1; echo exit: \$?"
```
Expected: `Connection refused` and exit 1 (L3 routing works across nodes).

## Cleanup

```bash
oc delete -k .
```
