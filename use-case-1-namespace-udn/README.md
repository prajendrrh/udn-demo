# Use Case 1: Namespace-Level UDN (Tenant Isolation)

Each namespace has its own **UserDefinedNetwork (UDN)**. Pods in tenant-a get IPs from `192.0.2.0/24`, pods in tenant-b from `198.51.100.0/24`. They are isolated from each other.

## Apply

```bash
oc apply -k .
```

Wait for pods: `oc get pods -n tenant-a -n tenant-b -o wide`

## Show UDN IPs

`oc get pods -o wide` shows the default cluster IP, not the UDN IP. To see UDN IPs, run:

```bash
# Tenant-a (expect 192.0.2.x)
for p in $(oc get pod -n tenant-a -l app=app-tenant-a -o jsonpath='{.items[*].metadata.name}'); do
  echo -n "$p: "; oc exec -n tenant-a $p -- ip -4 -o addr show 2>/dev/null | awk '{print $4}' | cut -d/ -f1
done

# Tenant-b (expect 198.51.100.x)
for p in $(oc get pod -n tenant-b -l app=app-tenant-b -o jsonpath='{.items[*].metadata.name}'); do
  echo -n "$p: "; oc exec -n tenant-b $p -- ip -4 -o addr show 2>/dev/null | awk '{print $4}' | cut -d/ -f1
done
```

## Test connectivity

- **Same tenant:** From one tenant-a pod, open a TCP connection to another tenant-a pod’s **UDN IP** (from above). You should get "Connection refused" (reachable).
- **Isolation:** From a tenant-a pod, try the same to a tenant-b **UDN IP**. You should get timeout (not reachable).

## Cleanup

```bash
oc delete -k .
```
