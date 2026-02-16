# Use Case 1: Namespace-Level UDN (Tenant Isolation)

Each namespace has its own **UserDefinedNetwork (UDN)**. Pods in tenant-a get IPs from `192.0.2.0/24`, pods in tenant-b from `198.51.100.0/24`. They are isolated from each other.

## Apply

```bash
oc apply -k .
```

Wait for pods:

```bash
oc get pods -n tenant-a
oc get pods -n tenant-b
```

## Show UDN IPs

`oc get pods -o wide` shows the default cluster IP, not the UDN IP. To list each pod and its UDN IP from the pod annotation `k8s.v1.cni.cncf.io/network-status` (requires `jq`):

```bash
echo "=== tenant-a (UDN 192.0.2.0/24) ==="
oc get pods -n tenant-a -l app=app-tenant-a -o json | jq -r '.items[] | .metadata.name as $n | ((.metadata.annotations["k8s.v1.cni.cncf.io/network-status"] // "[]") | fromjson)[0].ips[0] // "?" as $ip | "\($n): \($ip)"'

echo "=== tenant-b (UDN 198.51.100.0/24) ==="
oc get pods -n tenant-b -l app=app-tenant-b -o json | jq -r '.items[] | .metadata.name as $n | ((.metadata.annotations["k8s.v1.cni.cncf.io/network-status"] // "[]") | fromjson)[0].ips[0] // "?" as $ip | "\($n): \($ip)"'
```

Example output:

```
=== tenant-a (UDN 192.0.2.0/24) ===
app-7b8c9d-xk2m4: 192.0.2.2
app-7b8c9d-z9pqr: 192.0.2.3
=== tenant-b (UDN 198.51.100.0/24) ===
app-5d4e3f-ab12c: 198.51.100.2
app-5d4e3f-cd34e: 198.51.100.3
```

## Test connectivity

Use the UDN IPs from **Show UDN IPs** above. All checks use TCP (no `ping`).

**1. Within the same namespace (tenant-a → tenant-a)**  
From one tenant-a pod, connect to another tenant-a pod’s UDN IP. You should see "Connection refused" (reachable; nothing listens on port 80).

```bash
POD_A=$(oc get pod -n tenant-a -l app=app-tenant-a -o jsonpath='{.items[0].metadata.name}')
IP_A=$(oc get pods -n tenant-a -l app=app-tenant-a -o json | jq -r '.items[1].metadata.annotations["k8s.v1.cni.cncf.io/network-status"] | fromjson | .[0].ips[0]')
echo "From $POD_A to tenant-a UDN IP $IP_A (same namespace):"
oc exec -n tenant-a $POD_A -- bash -c "timeout 2 bash -c 'echo >/dev/tcp/'"$IP_A"'/80' 2>&1; echo exit: \$?"
```
Expected: `Connection refused` and exit 1 (traffic reaches the pod).

**2. Across namespaces (tenant-a → tenant-b)**  
From a tenant-a pod, connect to a tenant-b pod’s UDN IP. You should see timeout (isolated).

```bash
IP_B=$(oc get pods -n tenant-b -l app=app-tenant-b -o json | jq -r '.items[0].metadata.annotations["k8s.v1.cni.cncf.io/network-status"] | fromjson | .[0].ips[0]')
echo "From $POD_A to tenant-b UDN IP $IP_B (different namespace):"
oc exec -n tenant-a $POD_A -- bash -c "timeout 2 bash -c 'echo >/dev/tcp/'"$IP_B"'/80' 2>&1; echo exit: \$?"
```
Expected: timeout (e.g. exit 124) — no route to the other tenant.

## Cleanup

```bash
oc delete -k .
```
