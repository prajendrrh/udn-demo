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

Wait for pods to be Ready:

```bash
oc get pods -n team-platform-dev -n team-platform-staging -n team-platform-prod -o wide
```

## Verify

```bash
# CUDN status
oc get clusteruserdefinednetwork cudn-platform -o yaml

# NetworkAttachmentDefinitions created per selected namespace
oc get network-attachment-definitions -A | grep cudn-platform

# Pod IPs (all should be in 203.0.113.0/24)
oc get pods -n team-platform-dev -o wide
oc get pods -n team-platform-staging -o wide
oc get pods -n team-platform-prod -o wide
```

## Test steps

1. **Confirm all pod IPs are in the CUDN subnet** `203.0.113.0/24`.

2. **Cross-namespace connectivity (same CUDN)**
   - Get the pod IP in `team-platform-staging`:
     ```bash
     IP_STAGING=$(oc get pod -n team-platform-staging -l app=app-cudn -o jsonpath='{.items[0].status.podIP}')
     echo $IP_STAGING
     ```
   - From the dev pod, ping the staging pod:
     ```bash
     POD_DEV=$(oc get pod -n team-platform-dev -l app=app-cudn -o jsonpath='{.items[0].metadata.name}')
     oc exec -n team-platform-dev $POD_DEV -- ping -c 2 $IP_STAGING
     ```
   Expected: replies (pods on the shared CUDN can reach each other across namespaces).

3. **Optional:** Repeat from prod to dev to confirm full mesh on the CUDN.

## Cleanup

```bash
oc delete -k .
```
