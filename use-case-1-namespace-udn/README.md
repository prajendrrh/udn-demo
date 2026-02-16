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

Pods are given the `NET_RAW` capability so `ping` works from the container. If you already applied before and see "ping: permission denied", re-apply and restart the deployments so pods pick up the updated securityContext:

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
   ```bash
   # Get a pod name in tenant-a
   POD_A=$(oc get pod -n tenant-a -l app=app-tenant-a -o jsonpath='{.items[0].metadata.name}')
   # Get another pod's IP in tenant-a
   IP_A=$(oc get pod -n tenant-a -l app=app-tenant-a -o jsonpath='{.items[1].status.podIP}')
   oc exec -n tenant-a $POD_A -- ping -c 2 $IP_A
   ```
   Expected: replies (pods in same UDN can reach each other).

3. **Isolation across tenants**
   Get a tenant-b pod IP, then from a tenant-a pod try to ping it. You should see no reply or timeout (tenant isolation).
   ```bash
   IP_B=$(oc get pod -n tenant-b -l app=app-tenant-b -o jsonpath='{.items[0].status.podIP}')
   echo "Tenant-B pod IP: $IP_B"
   oc exec -n tenant-a $POD_A -- ping -c 2 $IP_B
   ```
   Expected: no replies (request timeouts or unreachable).

## Cleanup

```bash
oc delete -k .
```
