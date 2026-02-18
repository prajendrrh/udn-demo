# Use Case: NetworkPolicy with UDN

Demonstrates **NetworkPolicy** on **User-Defined Networks**: two namespaces (**np-udn-a** and **np-udn-b**) both attached to the **same** UDN via one **ClusterUserDefinedNetwork** (subnet 100.20.0.0/24), so pods can route to each other. A NetworkPolicy in np-udn-a allows ingress to the **server** pod only from namespace **np-udn-b**. The **client** pod in np-udn-b can reach the server; other namespaces cannot.

**Why both namespaces share one UDN:** Different UDNs are isolated at the network level (no routing between them). For NetworkPolicy to allow/deny traffic, the namespaces must be on the same network; the policy then restricts who can reach the server.

## What’s included

| Resource | Purpose |
|----------|--------|
| **np-udn-a**, **np-udn-b** | Namespaces with label `udn-shared: "true"` (selected by CUDN) and `np-udn-demo: "a"` / `"b"` (for NetworkPolicy) |
| **CUDN** `np-udn-shared` | One UDN 100.20.0.0/24 for both namespaces so they can reach each other |
| **Server** (np-udn-a) | Listens on port 8080 (agnhost serve-hostname); label `app=server` |
| **Client** (np-udn-b) | Label `app=client`; used to test connectivity to server |
| **NetworkPolicy** `allow-server-from-np-udn-b` | In np-udn-a: allow ingress to `app=server` only from namespaces with label `np-udn-demo: "b"`, port 8080 |

## Apply

```bash
oc apply -k use-case-network-policy-udn/
```

## Test

1. **Get server UDN IP** (100.20.0.x):
   ```bash
   oc get pod -n np-udn-a -l app=server -o json | jq -r '.items[0].metadata.annotations["k8s.v1.cni.cncf.io/network-status"]' | jq -r '.[] | select(.ips[0] | startswith("100.20.0.")) | .ips[0]'
   ```

2. **From client (np-udn-b)** — TCP to server (allowed by policy):
   ```bash
   CLIENT=$(oc get pod -n np-udn-b -l app=client -o jsonpath='{.items[0].metadata.name}')
   SERVER_IP=100.20.0.2   # use the IP from step 1
   oc exec -n np-udn-b $CLIENT -- timeout 2 bash -c "echo >/dev/tcp/$SERVER_IP/8080" 2>/dev/null && echo "Reachable" || echo "Timeout"
   ```
   Expected: **Reachable** (client in B is allowed to reach server in A).

3. **From another namespace** (e.g. default) — if you run a pod there and try to reach the server’s UDN IP, traffic should be **blocked** by the NetworkPolicy (only np-udn-b is allowed).

## Cleanup

```bash
oc delete -k use-case-network-policy-udn/
```

## References

- [Primary networks (UDN) – OpenShift 4.21](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/multiple_networks/primary-networks)
- [Network policy – OpenShift](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/networking/network-policy)
