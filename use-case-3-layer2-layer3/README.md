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

## Verify

```bash
oc get userdefinednetwork -n udn-layer2-demo
oc get userdefinednetwork -n udn-layer3-demo
```

## Cleanup

```bash
oc delete -k .
```
