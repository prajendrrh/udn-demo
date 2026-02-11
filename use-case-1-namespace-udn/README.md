# Use Case 1: Namespace-Level UDN (Tenant Isolation)

Each namespace has its own **UserDefinedNetwork (UDN)**. Pods (and VMs, if OpenShift Virtualization is used) in one namespace get IPs from that namespace’s subnet and cannot talk to workloads in other UDN namespaces without explicit routing/gateways.

## What this demonstrates

- **Tenant isolation**: `tenant-a` uses `192.0.2.0/24`, `tenant-b` uses `198.51.100.0/24`.
- **Namespace-scoped network**: UDN is defined per namespace; no cluster-wide CR.
- **Primary network**: The UDN is the primary network for the namespace (label `k8s.ovn.org/primary-user-defined-network: ""` on the namespace).

## Apply

```bash
oc apply -k .
```

## Verify

```bash
# Check UDN status
oc get userdefinednetwork -n tenant-a
oc get userdefinednetwork -n tenant-b

# After deploying pods, check IPs (example with a simple pod)
oc run nginx -n tenant-a --image=nginx --restart=Never --overrides='{"spec":{"containers":[{"name":"nginx","image":"nginx"}]}}'
oc get pod nginx -n tenant-a -o wide
```

## Cleanup

```bash
oc delete -k .
```
