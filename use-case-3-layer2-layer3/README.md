# Use Case 3: Layer2 vs Layer3 UDN Topologies

Compare **Layer2** (single L2 segment) and **Layer3** (per-node subnets, L3 routing) User-Defined Networks.

- **Layer2** (`udn-layer2-demo`): One namespace, one UDN. All pods on the **same subnet** `100.2.0.0/24`. Single logical switch; no routing between pods.
- **Layer3** (`udn-layer3-a`, `udn-layer3-b`): Two namespaces sharing one **Layer3 CUDN** (`200.1.0.0/22`). Each namespace’s pod is on a **different subnet** (per-node /26, e.g. `200.1.0.0/26` on one node, `200.1.1.0/26` on another). Traffic between the two pods **goes through L3 routing**, not the same L2 segment. One pod per namespace demonstrates this cross-subnet, routed connectivity.

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

Example output (Layer3 pods are on **different subnets**; e.g. 200.1.0.x vs 200.1.1.x — traffic between them goes through routing):

```
=== udn-layer2-demo (Layer2 100.2.0.0/24) ===
app-xxxxx-abc: 100.2.0.2
app-xxxxx-def: 100.2.0.3
=== udn-layer3-a (Layer3 CUDN, one subnet per node) ===
app-yyyyy-aaa: 200.1.0.2
=== udn-layer3-b (Layer3 CUDN, different subnet) ===
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

**2. Layer3: across namespaces, different subnets, via routing**  
The pod in `udn-layer3-a` and the pod in `udn-layer3-b` are on **different subnets** (each node has its own /26). From the pod in `udn-layer3-a`, connect to the pod in `udn-layer3-b`’s CUDN IP. Traffic goes **through L3 routing**, not a shared L2 segment. You should see "Connection refused" (reachable).

```bash
POD_L3A=$(oc get pod -n udn-layer3-a -l app=app-layer3 -o jsonpath='{.items[0].metadata.name}')
IP_L3B=$(oc get pods -n udn-layer3-b -l app=app-layer3 -o json | jq -r '.items[0].metadata.annotations["k8s.v1.cni.cncf.io/network-status"] | fromjson | map(select(.ips[0] | startswith("200.1."))) | .[0].ips[0]')
echo "From $POD_L3A (udn-layer3-a) to udn-layer3-b pod CUDN IP $IP_L3B:"
oc exec -n udn-layer3-a $POD_L3A -- bash -c "timeout 2 bash -c 'echo >/dev/tcp/'"$IP_L3B"'/80' 2>&1; echo exit: \$?"
```
Expected: `Connection refused` and exit 1 (reachable via L3 routing across different subnets).

## Advanced: Trace traffic flow (ovnkube-trace)

OpenShift provides **ovnkube-trace** to simulate how a packet flows through OVN logical switches and routers. Use it to see the difference between Layer2 (same logical switch) and Layer3 (different switches and a router in the path).

**1. Get the OVN-Kubernetes control-plane pod** (cluster-admin). The trace tool runs in the `ovnkube-cluster-manager` container:

```bash
OVN_POD=$(oc get pod -n openshift-ovn-kubernetes -l app=ovnkube-control-plane -o jsonpath='{.items[0].metadata.name}')
```

**2. Trace Layer2 flow (pod → pod, same namespace)**  
Traffic stays on one logical switch.

```bash
POD_L2_1=$(oc get pod -n udn-layer2-demo -l app=app-layer2 -o jsonpath='{.items[0].metadata.name}')
POD_L2_2=$(oc get pod -n udn-layer2-demo -l app=app-layer2 -o jsonpath='{.items[1].metadata.name}')
oc exec -n openshift-ovn-kubernetes $OVN_POD -c ovnkube-cluster-manager -- ovnkube-trace -src-namespace udn-layer2-demo -src $POD_L2_1 -dst-namespace udn-layer2-demo -dst $POD_L2_2 -tcp -dst-port 80
```

**3. Trace Layer3 flow (pod in -a → pod in -b)**  
Traffic crosses logical switches and goes through a logical router (L3).

```bash
POD_L3A=$(oc get pod -n udn-layer3-a -l app=app-layer3 -o jsonpath='{.items[0].metadata.name}')
POD_L3B=$(oc get pod -n udn-layer3-b -l app=app-layer3 -o jsonpath='{.items[0].metadata.name}')
oc exec -n openshift-ovn-kubernetes $OVN_POD -c ovnkube-cluster-manager -- ovnkube-trace -src-namespace udn-layer3-a -src $POD_L3A -dst-namespace udn-layer3-b -dst $POD_L3B -tcp -dst-port 80
```

Compare the output: Layer2 shows a single logical switch; Layer3 shows multiple switches and a router in the path. See [Tracing Openflow with ovnkube-trace](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/ovn-kubernetes_network_plugin/tracing-openflow-with-ovnkube-trace) (Red Hat 4.21).

## Cleanup

```bash
oc delete -k .
```
