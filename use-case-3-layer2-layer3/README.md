# Use Case 3: Layer2 vs Layer3 UDN Topologies

Compare **Layer2** (single L2 segment) and **Layer3** (per-node subnets, L3 routing) User-Defined Networks.

- **Layer2** (`udn-layer2-demo`): One namespace, one UDN. All pods on subnet `100.2.0.0/24`. Single logical switch.
- **Layer3** (`udn-layer3-a`, `udn-layer3-b`): Two namespaces sharing one **Layer3 CUDN** (`200.1.0.0/22`, per-node /26s). One pod per namespace so we can demonstrate **L3 connectivity across namespaces** without the "timed out waiting for annotations" issue of two pods in the same namespace.

Requires **cluster-admin** for the Layer3 CUDN.

## Apply

```bash
oc apply -k .
```

Wait for pods:

```bash
oc get pods -n udn-layer2-demo
oc get pods -n udn-layer3-a
oc get pods -n udn-layer3-b
```

## Show UDN IPs

`oc get pods -o wide` shows the default cluster IP. To list each pod and its UDN/CUDN IP from the annotation (requires `jq`):

```bash
echo "=== udn-layer2-demo (Layer2 100.2.0.0/24) ==="
oc get pods -n udn-layer2-demo -l app=app-layer2 -o json | jq -r '.items[] | .metadata.name + ": " + ((.metadata.annotations["k8s.v1.cni.cncf.io/network-status"] // "[]") | fromjson | map(select(.ips[0] | startswith("100.2.0."))) | .[0].ips[0] // "?")'

echo "=== udn-layer3-a (Layer3 CUDN 200.1.x.x) ==="
oc get pods -n udn-layer3-a -l app=app-layer3 -o json | jq -r '.items[] | .metadata.name + ": " + ((.metadata.annotations["k8s.v1.cni.cncf.io/network-status"] // "[]") | fromjson | map(select(.ips[0] | startswith("200.1."))) | .[0].ips[0] // "?")'

echo "=== udn-layer3-b (Layer3 CUDN 200.1.x.x) ==="
oc get pods -n udn-layer3-b -l app=app-layer3 -o json | jq -r '.items[] | .metadata.name + ": " + ((.metadata.annotations["k8s.v1.cni.cncf.io/network-status"] // "[]") | fromjson | map(select(.ips[0] | startswith("200.1."))) | .[0].ips[0] // "?")'
```

Example output:

```
=== udn-layer2-demo (Layer2 100.2.0.0/24) ===
app-xxxxx-abc: 100.2.0.2
app-xxxxx-def: 100.2.0.3
=== udn-layer3-a (Layer3 CUDN 200.1.x.x) ===
app-yyyyy-aaa: 200.1.0.2
=== udn-layer3-b (Layer3 CUDN 200.1.x.x) ===
app-yyyyy-bbb: 200.1.1.3
```

## Test connectivity

Use UDN/CUDN IPs from **Show UDN IPs**. Checks use TCP (bash `/dev/tcp`), no `ping`.

**1. Layer2: same namespace (pod → pod)**  
From one layer2 pod, connect to another layer2 pod’s UDN IP. You should see "Connection refused" (reachable).

```bash
POD_L2=$(oc get pod -n udn-layer2-demo -l app=app-layer2 -o jsonpath='{.items[0].metadata.name}')
IP_L2=$(oc get pods -n udn-layer2-demo -l app=app-layer2 -o json | jq -r '.items[1].metadata.annotations["k8s.v1.cni.cncf.io/network-status"] | fromjson | map(select(.ips[0] | startswith("100.2.0."))) | .[0].ips[0]')
echo "From $POD_L2 to layer2 UDN IP $IP_L2:"
oc exec -n udn-layer2-demo $POD_L2 -- bash -c "timeout 2 bash -c 'echo >/dev/tcp/'"$IP_L2"'/80' 2>&1; echo exit: \$?"
```
Expected: `Connection refused` and exit 1.

**2. Layer3: across namespaces (pod in -a → pod in -b)**  
From the pod in `udn-layer3-a`, connect to the pod in `udn-layer3-b`’s CUDN IP. You should see "Connection refused" (reachable). This demonstrates **L3 connectivity across namespaces** (and across nodes if the pods are on different nodes).

```bash
POD_L3A=$(oc get pod -n udn-layer3-a -l app=app-layer3 -o jsonpath='{.items[0].metadata.name}')
IP_L3B=$(oc get pods -n udn-layer3-b -l app=app-layer3 -o json | jq -r '.items[0].metadata.annotations["k8s.v1.cni.cncf.io/network-status"] | fromjson | map(select(.ips[0] | startswith("200.1."))) | .[0].ips[0]')
echo "From $POD_L3A (udn-layer3-a) to udn-layer3-b pod CUDN IP $IP_L3B:"
oc exec -n udn-layer3-a $POD_L3A -- bash -c "timeout 2 bash -c 'echo >/dev/tcp/'"$IP_L3B"'/80' 2>&1; echo exit: \$?"
```
Expected: `Connection refused` and exit 1 (L3 routing across namespaces).

## Cleanup

```bash
oc delete -k .
```

## Troubleshooting

**"timed out waiting for annotations" when creating Layer3 pods**  
Try: delete the stuck pod; confirm node subnets (`oc get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.metadata.annotations.k8s\.ovn\.org/node-subnets}{"\n"}{end}'`); check CNO/OVN-Kubernetes logs. With two namespaces and one pod each, each pod can land on a different node, which often avoids the timeout seen with two pods in the same namespace.
