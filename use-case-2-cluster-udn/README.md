# Use Case 2: ClusterUserDefinedNetwork (Multi-Namespace Shared Network)

A **ClusterUserDefinedNetwork (CUDN)** defines a single network that is available in multiple namespaces. Workloads in `team-platform-dev`, `team-platform-staging`, and `team-platform-prod` share the same L2 segment and can communicate by IP (e.g. for VM clustering or shared services).

## What this demonstrates

- **Cross-namespace connectivity**: Pods/VMs in different namespaces get IPs from the same subnet (`203.0.113.0/24`).
- **Namespace selection**: Only namespaces with label `team: platform` get the CUDN.
- **Cluster-admin resource**: ClusterUserDefinedNetwork is cluster-scoped; only cluster admins can create it.

## Apply

Requires cluster-admin (for the CUDN):

```bash
oc apply -k .
```

## Verify

```bash
# CUDN status
oc get clusteruserdefinednetwork cudn-platform -o yaml

# NetworkAttachmentDefinitions created per selected namespace
oc get network-attachment-definitions -A | grep cudn-platform
```

## Cleanup

```bash
oc delete -k .
```
