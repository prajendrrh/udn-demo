# Use Case 4: VMs on UDN + NetworkPolicy

Demonstrates **User-Defined Networks for OpenShift Virtualization** and **NetworkPolicy** for isolation.

## Why UDN for VMs?

- **Deterministic IPs**: VMs get stable IPs from the UDN subnet (`203.0.113.0/24`) instead of the default NAT behavior (e.g. 10.0.2.2/24).
- **Clustering & discovery**: VM clustering and backup/monitoring agents can use real guest IPs.
- **Persistent IPAM**: IPs survive VM reboot and live migration.

## What’s included

- Namespace `udn-vm-demo` with the primary-UDN label.
- **UserDefinedNetwork** `udn-vm` (Layer2, `203.0.113.0/24`, Persistent IPAM).
- **NetworkPolicy** `allow-same-namespace-only`: only allows ingress from pods in the same namespace (optional; remove from kustomization if not needed).

## Prerequisites

- OpenShift Virtualization installed.
- Create VMs in this namespace via the console or `VirtualMachine` manifests; they will automatically use the UDN.

## Apply

```bash
oc apply -k .
```

## Verify

```bash
oc get userdefinednetwork -n udn-vm-demo
oc get network-attachment-definitions -n udn-vm-demo
# After creating a VM:
oc get vmi -n udn-vm-demo
# VM guest IP will be from 203.0.113.0/24
```

## Cleanup

```bash
oc delete -k .
```
