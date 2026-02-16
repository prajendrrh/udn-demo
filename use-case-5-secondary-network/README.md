# Use Case 5: Secondary Network (OVN-Kubernetes NAD)

Demonstrates an **OVN-Kubernetes secondary network** using a `NetworkAttachmentDefinition` (NAD). Pods keep the **default cluster network** as primary and get an **additional interface** on the secondary L2 segment. This is different from primary UDN, where the UDN replaces the default network for the namespace.

## Primary vs secondary

| | Primary (UDN) | Secondary (this use case) |
|--|---------------|----------------------------|
| Default network | Replaced by UDN in labeled namespaces | Unchanged; all pods have default + secondary |
| Pod attachment | Automatic in UDN namespaces | Opt-in via `k8s.v1.cni.cncf.io/networks` |
| Use case | Tenant isolation, VM networking | Extra L2 segment for selected workloads |

## What’s included

- **Namespace** `secondary-net-demo` (no UDN label).
- **NetworkAttachmentDefinition** `l2-secondary`: OVN-Kubernetes layer2, subnet `10.100.200.0/24`, exclude `10.100.200.0/29`.
- **Deployment** `app-with-secondary`: pods annotated to attach to `secondary-net-demo/l2-secondary`.
- **MultiNetworkPolicy** example (commented out in kustomization): applies to the secondary network; requires `spec.useMultiNetworkPolicy: true` on the cluster Network operator.

## Prerequisites

- OVN-Kubernetes (default in OpenShift). Secondary NAD is supported on bare metal, IBM Power, IBM Z, LinuxONE, VMware vSphere, RHOSP.

## Apply

```bash
oc apply -k .
```

## Verify

```bash
# NAD exists
oc get network-attachment-definitions -n secondary-net-demo

# Pods and IPs
oc get pods -n secondary-net-demo -o wide

# Pod has two interfaces: default + secondary (check network-status)
oc get pod -n secondary-net-demo -l app=with-secondary -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{.metadata.annotations.k8s\.v1\.cni\.cncf\.io/network-status}{"\n"}{end}'
```

## Test steps

1. **Confirm two interfaces per pod**  
   Each pod should have default cluster IP and a secondary IP in `10.100.200.0/24`:
   ```bash
   oc get pod -n secondary-net-demo -l app=with-secondary -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.podIP}{"\t"}{.metadata.annotations.k8s\.v1\.cni\.cncf\.io/network-status}{"\n"}{end}' | head -1
   ```
   Parse the `network-status` JSON to see the secondary net (e.g. `10.100.200.x`).

2. **Ping on default network**  
   From one pod, ping another pod's **default** IP:
   ```bash
   POD1=$(oc get pod -n secondary-net-demo -l app=with-secondary -o jsonpath='{.items[0].metadata.name}')
   IP2=$(oc get pod -n secondary-net-demo -l app=with-secondary -o jsonpath='{.items[1].status.podIP}')
   oc exec -n secondary-net-demo $POD1 -- ping -c 2 $IP2
   ```
   Expected: replies (default network works).

3. **Ping on secondary network**  
   Use the `test-helper` pod (busybox with secondary net) to ping another pod's secondary IP. Get the secondary IP from `network-status` (e.g. `10.100.200.x`), then:
   ```bash
   POD_HELPER=$(oc get pod -n secondary-net-demo -l app=test-helper -o jsonpath='{.items[0].metadata.name}')
   # Replace 10.100.200.x with the secondary IP from oc get pod ... -o jsonpath='{.items[0].metadata.annotations.k8s\.v1\.cni\.cncf\.io/network-status}'
   oc exec -n secondary-net-demo $POD_HELPER -- ping -c 2 10.100.200.x
   ```
   Expected: replies on the secondary L2 segment.

## MultiNetworkPolicy (optional)

To use the example MultiNetworkPolicy:

1. Enable multi-network policy:  
   `oc patch network.operator.openshift.io cluster --type=merge -p '{"spec":{"useMultiNetworkPolicy":true}}'`
2. Add `multi-network-policy-example.yaml` to `kustomization.yaml` and apply again.

Policies target the secondary network via the annotation  
`k8s.v1.cni.cncf.io/policy-for: <namespace>/<network_attachment_definition_name>`.

## References

- [Secondary networks (OpenShift 4.21)](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/multiple_networks/secondary-networks)
- [Configuring multi-network policy](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/multiple_networks/secondary-networks#configuring-multi-network-policy)
