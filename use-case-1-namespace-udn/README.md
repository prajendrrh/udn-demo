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

# Pod list (note: oc get pods -o wide may show the default OVN-K cluster IP, not the UDN IP)
oc get pods -n tenant-a -o wide
oc get pods -n tenant-b -o wide
```

## Checking UDN IPs

`oc get pods -o wide` shows the default cluster network IP. To see the **actual UDN IPs** (192.0.2.0/24 for tenant-a, 198.51.100.0/24 for tenant-b), use one of the following.

**1. From inside the pod (most reliable)** — the primary interface in the pod is on the UDN:

```bash
# Tenant-a: UDN IPs (expect 192.0.2.x)
for p in $(oc get pod -n tenant-a -l app=app-tenant-a -o jsonpath='{.items[*].metadata.name}'); do
  echo -n "$p: "; oc exec -n tenant-a $p -- ip -4 -o addr show 2>/dev/null | awk '{print $4}' | cut -d/ -f1
done

# Tenant-b: UDN IPs (expect 198.51.100.x)
for p in $(oc get pod -n tenant-b -l app=app-tenant-b -o jsonpath='{.items[*].metadata.name}'); do
  echo -n "$p: "; oc exec -n tenant-b $p -- ip -4 -o addr show 2>/dev/null | awk '{print $4}' | cut -d/ -f1
done
```

If `ip` is not available in the image, use `hostname -I` (space-separated list of IPs; the first is usually the primary/UDN):

```bash
oc exec -n tenant-a $(oc get pod -n tenant-a -l app=app-tenant-a -o jsonpath='{.items[0].metadata.name}') -- hostname -I
oc exec -n tenant-b $(oc get pod -n tenant-b -l app=app-tenant-b -o jsonpath='{.items[0].metadata.name}') -- hostname -I
```

**2. Pod annotations** — primary UDN IP may appear in `network-status`:

```bash
oc get pod -n tenant-a -l app=app-tenant-a -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.metadata.annotations.k8s\.v1\.cni\.cncf\.io/network-status}{"\n"}{end}'
oc get pod -n tenant-b -l app=app-tenant-b -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.metadata.annotations.k8s\.v1\.cni\.cncf\.io/network-status}{"\n"}{end}'
```

Parse the JSON in `network-status` to get the IP for the primary/UDN interface.

**3. Node subnets** — confirm which UDN subnets are assigned to nodes:

```bash
oc get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.metadata.annotations.k8s\.ovn\.org/node-subnets}{"\n"}{end}'
```

Use the **UDN IPs** from (1) or (2) in the connectivity test steps below when `status.podIP` shows the default cluster IP.

## Test steps

Use **UDN IPs** from the "Checking UDN IPs" section for these tests. If you use `status.podIP` and see the default cluster IP, connectivity checks may target the wrong address.

1. **Confirm IPs are on the UDN subnets**  
   Run the "From inside the pod" loops under **Checking UDN IPs** and confirm tenant-a IPs are in `192.0.2.0/24` and tenant-b IPs in `198.51.100.0/24`.

2. **Connectivity within same tenant (same namespace)**  
   Set `IP_A` to the **UDN IP** of the second tenant-a pod (from the loop above or from `oc exec ... -- ip -4 -o addr show`), then from the first pod run TCP to that IP:
   ```bash
   POD_A=$(oc get pod -n tenant-a -l app=app-tenant-a -o jsonpath='{.items[0].metadata.name}')
   # Use the UDN IP of the other tenant-a pod (e.g. from "Checking UDN IPs" loop)
   IP_A="192.0.2.x"   # replace with actual UDN IP from the other tenant-a pod
   oc exec -n tenant-a $POD_A -- bash -c "timeout 2 bash -c 'echo >/dev/tcp/'"$IP_A"'/80' 2>&1; echo Exit: \$?"
   ```
   Expected: "Connection refused" (pod is reachable on the UDN).

3. **Isolation across tenants**  
   Set `IP_B` to a **UDN IP** of a tenant-b pod, then from tenant-a try TCP to it:
   ```bash
   IP_B="198.51.100.x"   # replace with actual UDN IP from a tenant-b pod
   oc exec -n tenant-a $POD_A -- bash -c "timeout 2 bash -c 'echo >/dev/tcp/'"$IP_B"'/80' 2>&1; echo Exit: \$?"
   ```
   Expected: timeout — tenant-b is not reachable from tenant-a.

## Cleanup

```bash
oc delete -k .
```
