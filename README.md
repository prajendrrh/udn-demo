# OpenShift Multiple Networks Demo (Primary UDN + Secondary)

A demo project showcasing **primary** (User-Defined Networks) and **secondary** network use cases on Red Hat OpenShift Container Platform (4.18+). Manifests follow the official docs:

- **Primary networks (UDN):** [Primary networks – OpenShift 4.21](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/multiple_networks/primary-networks)
- **Secondary networks:** [Secondary networks – OpenShift 4.21](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/multiple_networks/secondary-networks)

## Primary (UDN) vs secondary

| | Primary – User-Defined Networks | Secondary – NAD |
|--|---------------------------------|------------------|
| **CRs** | UserDefinedNetwork, ClusterUserDefinedNetwork | NetworkAttachmentDefinition |
| **Pod attachment** | Automatic in namespaces with UDN label | Opt-in via `k8s.v1.cni.cncf.io/networks` |
| **Default network** | In UDN namespaces, primary is the UDN (replaces default) | Default unchanged; pod gets default + extra interface(s) |
| **Typical use** | Tenant isolation, VM networking, deterministic IPs | Extra L2/L3 segment for selected workloads |

## What are User-Defined Networks (primary)?

User-Defined Networks extend the OVN-Kubernetes plugin to provide:

- **Custom Layer 2 and Layer 3 segments** with default isolation
- **Workload and tenant isolation** without relying only on NetworkPolicies
- **Deterministic IP addressing** for pods and VMs (critical for OpenShift Virtualization)
- **Cluster-wide or namespace-scoped** networks via ClusterUserDefinedNetwork (CUDN) or UserDefinedNetwork (UDN)

## What you need to use this demo

Before committing or sharing, ensure the following are clear for anyone running the demo:

| Requirement | Details |
|-------------|---------|
| **Cluster** | OpenShift Container Platform **4.18 or later** (UDN GA from 4.18). Nodes must use **cgroupv2** (default on RHCOS). |
| **Access** | `oc` CLI; **cluster-admin** for CUDN (use cases 2, 7 CUDN) and BGP/route-advertisement config; **project-admin** is enough for namespace UDN (1, 3, 4). |
| **Use case 4 (Overlapping IPs)** | No extra operators; two UDNs with the same subnet show overlapping pod IPs across namespaces. |
| **Use case 5 (multihoming)** | No extra operators; primary UDN + secondary NAD on same pod. Optional **MultiNetworkPolicy** requires `useMultiNetworkPolicy` (see use-case-5 README). |
| **Use case 6 (BGP)** | **Bare metal**; BGP/FRR enabled on CNO; edit **FRRConfiguration** with your BGP peer/ASN before applying. |
| **Use case 7 (Route ads)** | **Bare metal**; CNO must have **FRR** and **routeAdvertisements: Enabled**; edit **neighbor address** in `frrconfiguration-receive-all.yaml` (e.g. route reflector IP) before applying. |

**Before you apply:**

1. **Use cases 1–5:** No cluster config changes required; apply with `oc apply -k <use-case-dir>/`.
2. **Use case 6:** Enable FRR on CNO (see `use-case-6-bgp-routing/README.md`), then edit the FRRConfiguration neighbor/ASN and apply.
3. **Use case 7:** Enable FRR and route advertisements on CNO (see `use-case-7-route-advertisements/README.md`), edit the BGP neighbor in `frrconfiguration-receive-all.yaml`, then apply (FRRConfiguration before RouteAdvertisements).

**Optional before commit:** If your team uses a specific BGP peer IP or ASN, document it (e.g. in a README note or `.env.example`) so others know what to replace.

## Prerequisites (summary)

- OpenShift Container Platform **4.18 or later** (UDN is GA from 4.18)
- Cluster nodes using **cgroupv2** (required for UDN)
- For VM use cases: **OpenShift Virtualization** installed (Layer2 and Localnet only)

## Use Cases in This Demo

Each use case creates **namespaces, network resources, and pods** (Deployments). Each has a **Test steps** section in its README so you can verify connectivity and behavior.

| Use Case | Type | Description | Path |
|----------|------|-------------|------|
| **1. Namespace UDN** | Primary | Single-tenant isolation with a UserDefinedNetwork per namespace | `use-case-1-namespace-udn/` |
| **2. Cluster UDN** | Primary | Multi-namespace shared network with ClusterUserDefinedNetwork | `use-case-2-cluster-udn/` |
| **3. Layer2 vs Layer3** | Primary | Layer2 and Layer3 UDN topologies | `use-case-3-layer2-layer3/` |
| **4. Overlapping pod IPs** | Primary | Two UDNs with the same subnet; pods in different namespaces can have the same UDN IP | `use-case-4-vm-and-policies/` |
| **5. Multihoming (primary UDN + secondary)** | Primary + Secondary | One primary UDN and one secondary NAD per pod; two interfaces | `use-case-5-secondary-network/` |
| **6. BGP routing** | Advanced | FRR-K8s, FRRConfiguration, custom BGP | `use-case-6-bgp-routing/` |
| **7. Route advertisements** | Advanced | Advertise pod/CUDN routes via BGP, RouteAdvertisements CR | `use-case-7-route-advertisements/` |

## Quick Start

Apply a use case (requires cluster-admin for CUDN, or project-admin for namespace UDN):

```bash
# Use case 1: Namespace-level UDN (tenant isolation)
oc apply -k use-case-1-namespace-udn/

# Use case 2: Cluster UDN (multi-namespace)
oc apply -k use-case-2-cluster-udn/

# Use case 3: Layer2 and Layer3 UDN
oc apply -k use-case-3-layer2-layer3/

# Use case 4: Overlapping pod IPs (two UDNs, same subnet)
oc apply -k use-case-4-vm-and-policies/

# Use case 5: Multihoming (primary UDN + secondary NAD)
oc apply -k use-case-5-secondary-network/

# Use case 6: BGP routing (enable FRR first; see use-case-6-bgp-routing/README.md)
oc apply -k use-case-6-bgp-routing/

# Use case 7: Route advertisements (enable FRR + routeAdvertisements; see use-case-7-route-advertisements/README.md)
oc apply -k use-case-7-route-advertisements/
```

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
