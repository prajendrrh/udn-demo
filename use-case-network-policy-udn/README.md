# Use Case: NetworkPolicy with UDN

One namespace (**np-udn-demo**) on one UDN (100.20.0.0/24). **Server** (nginx, port 80) and **client** pods run in the same namespace. A **NetworkPolicy** allows ingress to the server only from pods with label **app=client**, so only the client pod can reach the server.

## What’s included

| Resource | Purpose |
|----------|--------|
| **np-udn-demo** | Namespace with primary UDN (100.20.0.0/24) |
| **Server** | nginx:alpine, listens on **port 80**, label `app=server` |
| **Client** | Pod with label `app=client`; only this pod can reach the server |
| **NetworkPolicy** `allow-server-from-client` | Allow ingress to `app=server` only from pods with `app=client`, port 80 |

## Apply

```bash
oc apply -k use-case-network-policy-udn/
```

Wait for pods: `oc get pods -n np-udn-demo`

## Test

1. **Get server UDN IP** (100.20.0.x):
   ```bash
   SERVER_IP=$(oc get pod -n np-udn-demo -l app=server -o json | jq -r '.items[0].metadata.annotations["k8s.v1.cni.cncf.io/network-status"]' | jq -r '.[] | select(.ips[0] | startswith("100.20.0.")) | .ips[0]')
   echo $SERVER_IP
   ```

2. **From client pod — TCP to server (port 80, allowed by policy):**
   ```bash
   CLIENT=$(oc get pod -n np-udn-demo -l app=client -o jsonpath='{.items[0].metadata.name}')
   oc exec -n np-udn-demo $CLIENT -- timeout 2 bash -c "echo >/dev/tcp/$SERVER_IP/80" 2>/dev/null && echo "Reachable" || echo "Timeout"
   ```
   Expected: **Reachable**.

3. **Optional — from client, curl the server:**
   ```bash
   oc exec -n np-udn-demo $CLIENT -- curl -s -m 2 http://$SERVER_IP/
   ```
   Expected: nginx HTML (or "Reachable" if curl not available).

## Cleanup

```bash
oc delete -k use-case-network-policy-udn/
```

## References

- [Primary networks (UDN) – OpenShift 4.21](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/multiple_networks/primary-networks)
- [Network policy – OpenShift](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/networking/network-policy)
