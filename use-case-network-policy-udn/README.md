# NetworkPolicy with UDN

One namespace (**np-udn-demo**) on one UDN (100.20.0.0/24). **Server** and **client** pods run in the same namespace. By default, same-namespace communication is allowed in UDN; to actually demonstrate NetworkPolicy we use:

1. **Default-deny all ingress** — no pod can receive traffic from any other pod.
2. **Allow only client → server** — so only the client pod can reach the server on port 9376.

Without the allow policy, even the client could not reach the server; with both policies, only client→server is permitted.

## What’s included

| Resource | Purpose |
|----------|--------|
| **np-udn-demo** | Namespace with primary UDN (100.20.0.0/24) |
| **Server** | agnhost serve-hostname on **port 9376**, label `app=server` |
| **Client** | agnhost (has curl); label `app=client`; only this pod can reach the server |
| **NetworkPolicy** `default-deny-ingress` | Deny all ingress to all pods in the namespace |
| **NetworkPolicy** `allow-server-from-client` | Allow ingress to `app=server` only from pods with `app=client`, port 9376 |

## Apply

Use **`oc apply -k`** (same as use-case-8) so resources are applied in the right order (namespaces first, then UDN, then deployments and network policy). Do **not** use `oc apply -f` on the directory.

```bash
oc apply -k use-case-network-policy-udn/
```

Wait for pods: `oc get pods -n np-udn-demo` (both server and client should be Running).

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

3. **From client pod — reach server (port 9376, allowed by allow policy):**  
   With default-deny + allow-from-client, only the client can reach the server.
   ```bash
   CLIENT=$(oc get pods -n np-udn-demo -l app=client -o json | jq -r '.items[0].metadata.name // empty')
   oc exec -n np-udn-demo $CLIENT -- curl -s -m 3 http://$SERVER_IP:9376
   ```
   Expected: hostname string (e.g. `server-xxxxx`). Without the allow policy, this would fail (denied by default-deny-ingress).

4. **Optional — TCP check instead of curl:**
   ```bash
   oc exec -n np-udn-demo $CLIENT -- timeout 2 bash -c "echo >/dev/tcp/$SERVER_IP/9376" 2>/dev/null && echo "Reachable" || echo "Timeout"
   ```

5. **Optional — verify deny:** From any pod that is not the client (e.g. a temporary `oc run` pod in the same namespace), `curl` or TCP to `$SERVER_IP:9376` should fail or timeout, because only the client is allowed by the allow policy.

## Cleanup

```bash
oc delete -k use-case-network-policy-udn/
```

## References

- [Primary networks (UDN) – OpenShift 4.21](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/multiple_networks/primary-networks)
- [Network policy – OpenShift](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/networking/network-policy)
