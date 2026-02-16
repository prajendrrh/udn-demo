# Use Case 4: VMs on UDN + NetworkPolicy

Demonstrates **User-Defined Networks for OpenShift Virtualization** and **NetworkPolicy** for isolation.

## Why UDN for VMs?

- **Deterministic IPs**: VMs get stable IPs from the UDN subnet (`203.0.113.0/24`) instead of the default NAT behavior (e.g. 10.0.2.2/24).
- **Clustering & discovery**: VM clustering and backup/monitoring agents can use real guest IPs.
- **Persistent IPAM**: IPs survive VM reboot and live migration.

## What's included

- Namespace `udn-vm-demo` with the primary-UDN label.
- **UserDefinedNetwork** `udn-vm` (Layer2, `203.0.113.0/24`, Persistent IPAM).
- **NetworkPolicy** `allow-same-namespace-only`: only allows ingress from pods in the same namespace (optional; remove from kustomization if not needed).
- **Deployment** `app`: pods on the UDN for testing connectivity and policy.

## Prerequisites

- OpenShift Virtualization installed (for VMs). Pods can run without it.
- Create VMs in this namespace via the console or `VirtualMachine` manifests; they will automatically use the UDN.

## Apply

```bash
oc apply -k .
```

Wait for pods:

```bash
oc get pods -n udn-vm-demo -o wide
```

## Verify

```bash
oc get userdefinednetwork -n udn-vm-demo
oc get network-attachment-definitions -n udn-vm-demo
oc get networkpolicies -n udn-vm-demo
# After creating a VM:
oc get vmi -n udn-vm-demo
# VM guest IP will be from 203.0.113.0/24
```

## Test steps

1. **Confirm pod IPs are on the UDN** (`203.0.113.0/24`)
   ```bash
   oc get pods -n udn-vm-demo -o wide
   ```

2. **Same-namespace connectivity (allowed by NetworkPolicy)**
   ```bash
   POD1=$(oc get pod -n udn-vm-demo -l app=app-udn-vm -o jsonpath='{.items[0].metadata.name}')
   IP2=$(oc get pod -n udn-vm-demo -l app=app-udn-vm -o jsonpath='{.items[1].status.podIP}')
   oc exec -n udn-vm-demo $POD1 -- ping -c 2 $IP2
   ```
   Expected: replies (ingress from same namespace is allowed).

3. **NetworkPolicy effect**: From a pod in another namespace (e.g. `default`), try to reach a pod IP in `udn-vm-demo`. Traffic should be denied by the policy (no reply or connection refused), demonstrating that only same-namespace ingress is allowed.

## Cleanup

```bash
oc delete -k .
```
