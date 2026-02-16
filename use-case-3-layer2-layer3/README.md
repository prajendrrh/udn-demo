# Use Case 3: Layer2 vs Layer3 UDN Topologies

Compare **Layer2** and **Layer3** User-Defined Networks.

## Layer2 (`udn-layer2-demo`)

- **Single logical switch** shared by all nodes.
- All pods get IPs from the same subnet (`192.0.2.0/24`).
- Suited for: VM clustering, L2 discovery, workloads that expect a flat L2 segment.

## Layer3 (`udn-layer3-demo`)

- **One L2 segment per node**, each with a different subnet; L3 routing connects them.
- The CIDR (`198.51.100.0/24`) is split into per-node subnets automatically (or by `hostSubnet`).
- Suited for: L3-only designs, larger scale, avoiding a single large L2 domain.

## Apply

```bash
oc apply -k .
```

Wait for pods:

```bash
oc get pods -n udn-layer2-demo -o wide
oc get pods -n udn-layer3-demo -o wide
```

## Verify

```bash
oc get userdefinednetwork -n udn-layer2-demo
oc get userdefinednetwork -n udn-layer3-demo
```

## Test steps

1. **Layer2: same subnet**
   - Pods in `udn-layer2-demo` should have IPs in `192.0.2.0/24`.
   - From one pod, ping another in the same namespace:
     ```bash
     POD1=$(oc get pod -n udn-layer2-demo -l app=app-layer2 -o jsonpath='{.items[0].metadata.name}')
     IP2=$(oc get pod -n udn-layer2-demo -l app=app-layer2 -o jsonpath='{.items[1].status.podIP}')
     oc exec -n udn-layer2-demo $POD1 -- ping -c 2 $IP2
     ```
   Expected: replies.

2. **Layer3: per-node subnets**
   - Pods in `udn-layer3-demo` get IPs from `198.51.100.0/24` but split by node (e.g. 198.51.100.0/24, 198.51.100.1/24, …). Check:
     ```bash
     oc get pods -n udn-layer3-demo -o wide
     ```
   - Ping between two pods (same or different node); L3 routing should allow connectivity:
     ```bash
     POD1=$(oc get pod -n udn-layer3-demo -l app=app-layer3 -o jsonpath='{.items[0].metadata.name}')
     IP2=$(oc get pod -n udn-layer3-demo -l app=app-layer3 -o jsonpath='{.items[1].status.podIP}')
     oc exec -n udn-layer3-demo $POD1 -- ping -c 2 $IP2
     ```
   Expected: replies (L3 routing between nodes).

## Cleanup

```bash
oc delete -k .
```
