# Use Case: NetworkPolicy with UDN

One namespace (**np-udn-demo**) on one UDN (100.20.0.0/24). **Server** (nginx, port 80) and **client** pods run in the same namespace. A **NetworkPolicy** allows ingress to the server only from pods with label **app=client**, so only the client pod can reach the server.

## What’s included

| Resource | Purpose |
|----------|--------|
| **np-udn-demo** | Namespace with primary UDN (100.20.0.0/24) |
| **Server** | Same image as client, listens on **port 8080** (nc), label `app=server` |
| **Client** | Pod with label `app=client`; only this pod can reach the server |
| **NetworkPolicy** `allow-server-from-client` | Allow ingress to `app=server` only from pods with `app=client`, port 8080 |

## Apply

```bash
oc apply -k use-case-network-policy-udn/
```

Wait for pods: `oc get pods -n np-udn-demo`

## Test

1. **Ensure the server pod exists and is Running:**
   ```bash
   oc get pods -n np-udn-demo -l app=server
   ```
   If the list is empty or the pod is not `Running`, wait for the deployment (or run `oc get pods -n np-udn-demo` and fix any ImagePullBackOff / CrashLoopBackOff).

2. **Get server UDN IP** (safe when no pods yet — returns empty instead of error):
   ```bash
   SERVER_IP=$(oc get pods -n np-udn-demo -l app=server -o json | jq -r '.items[0] | if . == null then empty else ( (.metadata.annotations["k8s.v1.cni.cncf.io/network-status"] // "[]") | fromjson | map(select(.ips[0] | startswith("100.20.0."))) | .[0].ips[0] ) // .status.podIP end')
   echo "Server IP: $SERVER_IP"
   ```
   This uses the network-status annotation (UDN 100.20.0.x) when present, otherwise `status.podIP`. If `SERVER_IP` is empty, the server pod is not ready yet.

3. **From client pod — TCP to server (port 8080, allowed by policy):**
   ```bash
   CLIENT=$(oc get pods -n np-udn-demo -l app=client -o json | jq -r '.items[0].metadata.name // empty')
   oc exec -n np-udn-demo $CLIENT -- timeout 2 bash -c "echo >/dev/tcp/$SERVER_IP/8080" 2>/dev/null && echo "Reachable" || echo "Timeout"
   ```
   Expected: **Reachable**.

4. **Optional — from client, curl the server** (if the image has curl and server responded on HTTP):
   ```bash
   oc exec -n np-udn-demo $CLIENT -- curl -s -m 2 http://$SERVER_IP:8080/ || true
   ```

## Cleanup

```bash
oc delete -k use-case-network-policy-udn/
```

## References

- [Primary networks (UDN) – OpenShift 4.21](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/multiple_networks/primary-networks)
- [Network policy – OpenShift](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/networking/network-policy)
