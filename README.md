# OpenShift Multiple Networks Demo (Primary UDN + Secondary)

A demo project showcasing **primary** (User-Defined Networks) and **secondary** network use cases on Red Hat OpenShift Container Platform (4.18+). Manifests follow the official docs:

- **Primary networks (UDN):** [Primary networks – OpenShift 4.21](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/multiple_networks/primary-networks)
- **Secondary networks:** [Secondary networks – OpenShift 4.21](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/multiple_networks/secondary-networks)

## What are User-Defined Networks (primary)?

User-Defined Networks extend the OVN-Kubernetes plugin to provide:

- **Custom Layer 2 and Layer 3 segments** with default isolation
- **Workload and tenant isolation** without relying only on NetworkPolicies
- **Deterministic IP addressing** for pods and VMs (critical for OpenShift Virtualization)
- **Cluster-wide or namespace-scoped** networks via ClusterUserDefinedNetwork (CUDN) or UserDefinedNetwork (UDN)

## Prerequisites

- OpenShift Container Platform **4.18 or later** (UDN is GA from 4.18). Nodes must use **cgroupv2** (default on RHCOS).
- `oc` CLI. **cluster-admin** is required for CUDN and BGP/route-advertisement; **project-admin** is enough for namespace-scoped UDN.
- For VM use cases: **OpenShift Virtualization** installed (Layer2 and Localnet only).

## Use cases

Each use case has its own directory with a README (apply steps, test, cleanup). Use `oc apply -k <directory>/`; do not use `oc apply -f` on a directory.

---

**Tenant isolation** — `use-case-tenant-isolation/`  
One UserDefinedNetwork per namespace (e.g. tenant-a, tenant-b). Pods get UDN IPs; namespaces are isolated.

**Cluster UDN** — `use-case-cluster-udn/`  
Multi-namespace shared network with ClusterUserDefinedNetwork. Several namespaces share one CUDN and can reach each other.

**Layer2 vs Layer3** — `use-case-layer2-layer3/`  
Layer2 and Layer3 UDN topologies in separate namespaces.

**Overlapping pod IPs** — `use-case-overlapping-pod-ips/`  
Two UDNs with the same subnet; pods in different namespaces can have the same UDN IP (e.g. 100.4.0.2 in both). No conflict—each UDN is a separate logical network.

**Multihoming (primary UDN + secondary UDN)** — `use-case-multi-homing/`  
One primary UDN and one secondary UDN per pod; pod gets two interfaces. Optional MultiNetworkPolicy requires `useMultiNetworkPolicy` (see use-case README).

**NetworkPolicy with UDN** — `use-case-network-policy-udn/`  
One namespace on a UDN; server and client pods. NetworkPolicy allows ingress to the server only from the client pod.

**Services: default network vs UDN** — `use-case-services-default-vs-udn/`  
One namespace on the **default** network and one on a **single UDN**. Shows that Services on the default network are reachable only from default-network pods, Services on a UDN only from that UDN’s pods, and that UDN pods can still reach KAPI and DNS.

**Services: two UDNs (BLUE and RED)** — `use-case-8-services-isolated/`  
Two namespaces, each on a **different UDN** (BLUE and RED; no default network in this demo). Only pods on BLUE can reach Service blue; only pods on RED can reach Service red. Shows service isolation **between** UDNs.

**BGP integration** — `use-case-bgp-integration/`  
Advanced: FRR on a VM, enable FRR/route-advertisement in OpenShift, UDN + FRRConfiguration + RouteAdvertisements. Cluster advertises UDN subnet to the bastion so you can reach UDN pods from the VM.

---

## Quick Start

Apply any use case (directory path as above):

```bash
oc apply -k use-case-tenant-isolation/
oc apply -k use-case-cluster-udn/
oc apply -k use-case-layer2-layer3/
oc apply -k use-case-overlapping-pod-ips/
oc apply -k use-case-multi-homing/
oc apply -k use-case-network-policy-udn/
oc apply -k use-case-services-default-vs-udn/
oc apply -k use-case-8-services-isolated/
oc apply -k use-case-bgp-integration/
```

For BGP integration, configure FRR on the VM and enable FRR+RA in OpenShift first; see `use-case-bgp-integration/README.md`.

## Important: Namespace Label

Namespaces that use a **primary** User-Defined Network must have this label **at creation time** (it cannot be added later):

```yaml
metadata:
  labels:
    k8s.ovn.org/primary-user-defined-network: ""
```

All manifests in this repo create namespaces with this label where needed.

## Limitations (from official docs)

- **Namespace and network first**: Create the namespace and UDN/CUDN before creating pods; you cannot add the label or assign an existing namespace to a UDN later.
- **DNS**: Pod DNS resolves to the pod’s IP on the **default** network, not the UDN IP (services and external DNS work as usual). UDN pods can still reach the Kubernetes API and cluster DNS.
- **Default network services**: UDN pods are isolated from the default network; e.g. the image registry and some `openshift-*.svc` services may be inaccessible (source-to-image and `oc new-app` from Git can be affected).
- **Immutable CRs**: ClusterUserDefinedNetwork and UserDefinedNetwork cannot be modified after creation.
- **Best practices**: Do not select the `default` or `openshift-*` namespaces with a CUDN. Avoid an empty `namespaceSelector` (it selects all namespaces).

For the full list, see [Limitations of a user-defined network](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/multiple_networks/primary-networks) in the 4.21 docs.

## Allowing default-network access to UDN pods

To allow pods on the default network (e.g. Prometheus, registry) to reach a UDN pod, use the pod annotation `k8s.ovn.org/open-default-ports`:

```yaml
metadata:
  annotations:
    k8s.ovn.org/open-default-ports: |
      - protocol: tcp
        port: 8080
```

Ports are opened on the pod’s **default network IP**, not the UDN IP. See [Opening default network ports on user-defined network pods](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/multiple_networks/primary-networks#opening-default-network-ports-on-user-defined-network-pods_udn).

## References

- [Primary networks – OpenShift 4.21](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/multiple_networks/primary-networks) – UserDefinedNetwork, ClusterUserDefinedNetwork
- [Secondary networks – OpenShift 4.21](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/multiple_networks/secondary-networks) – NAD, pod attachment, MultiNetworkPolicy
- [BGP routing – OpenShift 4.21](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/advanced_networking/bgp-routing) – FRR-K8s, FRRConfiguration
- [Route advertisements – OpenShift 4.21](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/advanced_networking/route-advertisements) – Advertise pod/CUDN routes via BGP
- [User defined networks in Red Hat OpenShift Virtualization](https://www.redhat.com/en/blog/user-defined-networks-red-hat-openshift-virtualization)
- [OpenShift Examples – UDN](https://examples.openshift.pub/networking/udn/)
