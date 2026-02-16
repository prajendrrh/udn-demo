# Use Case 1: Namespace-Level UDN (Tenant Isolation)

Each namespace has its own **UserDefinedNetwork (UDN)**. Pods (and VMs, if OpenShift Virtualization is used) in one namespace get IPs from that namespace's subnet and cannot talk to workloads in other UDN namespaces without explicit routing/gateways.

## What this demonstrates

- **Tenant isolation**: `tenant-a` uses `192.0.2.0/24`, `tenant-b` uses `198.51.100.0/24`.
- **Namespace-scoped network**: UDN is defined per namespace; no cluster-wide CR.
- **Primary network**: The UDN is the primary network for the namespace (label `k8s.ovn.org/primary-user-defined-network: ""` on the namespace).

## Apply

```bash
oc apply -k .
```

Pods use **UBI 9 minimal** (`registry.redhat.io/ubi9/ubi-minimal:9.4`). Connectivity is tested via **TCP** (bash `/dev/tcp`) so no `NET_RAW` capability is required. If you already applied, re-apply and restart to pick up changes:

```bash
oc apply -k .
oc rollout restart deployment/app -n tenant-a && oc rollout restart deployment/app -n tenant-b
```

Wait for pods to be Ready:

```bash
oc get pods -n tenant-a -w
oc get pods -n tenant-b -w
```

## Verify

```bash
# UDN status
oc get userdefinednetwork -n tenant-a
oc get userdefinednetwork -n tenant-b

# Pods and their IPs (should be in 192.0.2.0/24 and 198.51.100.0/24)
oc get pods -n tenant-a -o wide
oc get pods -n tenant-b -o wide
```

## Test steps

1. **Confirm IPs are on the UDN subnets**
   - tenant-a pods: IPs in `192.0.2.0/24`
   - tenant-b pods: IPs in `198.51.100.0/24`

2. **Connectivity within same tenant (same namespace)**  
   Use TCP to test reachability (no listener needed; "Connection refused" means the host is reachable).
   ```bash
   POD_A=$(oc get pod -n tenant-a -l app=app-tenant-a -o jsonpath='{.items[0].metadata.name}')
   IP_A=$(oc get pod -n tenant-a -l app=app-tenant-a -o jsonpath='{.items[1].status.podIP}')
   oc exec -n tenant-a $POD_A -- bash -c "timeout 2 bash -c 'echo >/dev/tcp/'"$IP_A"'/80' 2>&1; echo Exit: \$?"
   ```
   Expected: "Connection refused" or "connect: Connection refused" and exit 1 — that means the pod is **reachable** (nothing listens on port 80). Timeout would mean unreachable.

3. **Isolation across tenants**  
   From a tenant-a pod, try TCP to a tenant-b pod IP. You should see **timeout** (no route / isolation).
   ```bash
   IP_B=$(oc get pod -n tenant-b -l app=app-tenant-b -o jsonpath='{.items[0].status.podIP}')
   echo "Tenant-B pod IP: $IP_B"
   oc exec -n tenant-a $POD_A -- bash -c "timeout 2 bash -c 'echo >/dev/tcp/'"$IP_B"'/80' 2>&1; echo Exit: \$?"
   ```
   Expected: timeout (e.g. "timed out" or exit 124) — tenant-b is not reachable from tenant-a.

## Cleanup

```bash
oc delete -k .
```
