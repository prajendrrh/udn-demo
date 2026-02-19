# Services: default network vs UDN

This use case shows how **Kubernetes Services** behave when you have both the **default network** and **one UDN**: one namespace on the default network (default-demo) and one on a UDN (udn-demo). It documents the rules and provides a small demo. (For service isolation **between two UDNs**, see `use-case-8-services-in-udn/`.)

---

## Important points (UDN + Services)

- **Services are namespace-scoped** (as per Kubernetes).
- **Accessibility:**
  - **Services on the default network** are accessible only from namespaces on the default network.
  - **Services on a UDN** are accessible only from namespaces connected to that UDN.
- **One service CIDR per cluster**, shared by the default network and all UDNs.
- **Exceptions:** Pods on a UDN can still access **Kubernetes API (KAPI)** and **DNS** on the default network.
- **Full support for:**
  - ClusterIP, NodePort, LoadBalancer services
  - OVN-Kubernetes–networked and host-networked endpoints
  - Layer 2 and Layer 3 primary UDNs
  - Shared and local gateway mode

---

## What the demo does

| Namespace     | Network   | Subnet        | Service   |
|--------------|-----------|---------------|-----------|
| default-demo | default   | (cluster pod CIDR) | default-svc |
| udn-demo     | UDN       | 100.30.0.0/24 | udn-svc   |

- **default-demo:** No UDN label → pods on the **default network**. Service **default-svc** is reachable only from pods on the default network.
- **udn-demo:** Primary UDN → pods on **100.30.0.0/24**. Service **udn-svc** is reachable only from pods in udn-demo (same UDN). Pods in udn-demo can still reach KAPI and cluster DNS.

---

## Apply

```bash
oc apply -k use-case-udn-services/
```

Wait for pods: `oc get pods -n default-demo` and `oc get pods -n udn-demo`.

---

## Test (accessibility rules)

1. **Default pod → default Service (same network) — should succeed**
   ```bash
   POD=$(oc get pod -n default-demo -l app=default-app -o jsonpath='{.items[0].metadata.name}')
   oc exec -n default-demo $POD -- curl -s -m 3 http://default-svc:9376
   ```
   Expected: response from the server (e.g. hostname).

2. **Default pod → UDN Service (different network) — should fail**
   ```bash
   oc exec -n default-demo $POD -- curl -s -m 3 http://udn-svc.udn-demo.svc.cluster.local:9376 || echo "Expected: unreachable"
   ```
   Expected: timeout or failure. Default network cannot reach UDN services.

3. **UDN pod → UDN Service (same network) — should succeed**
   ```bash
   UDN_POD=$(oc get pod -n udn-demo -l app=udn-app -o jsonpath='{.items[0].metadata.name}')
   oc exec -n udn-demo $UDN_POD -- curl -s -m 3 http://udn-svc:9376
   ```
   Expected: response. Only pods on this UDN can reach udn-svc.

4. **UDN pod → default Service (different network) — should fail**
   ```bash
   oc exec -n udn-demo $UDN_POD -- curl -s -m 3 http://default-svc.default-demo.svc.cluster.local:9376 || echo "Expected: unreachable"
   ```
   Expected: timeout or failure. UDN cannot reach default-network services.

5. **UDN pod → Kubernetes API and DNS (exception) — should succeed**
   ```bash
   oc exec -n udn-demo $UDN_POD -- curl -s -k -m 3 https://kubernetes.default.svc.cluster.local/version
   oc exec -n udn-demo $UDN_POD -- nslookup kubernetes.default.svc.cluster.local
   ```
   Expected: KAPI and DNS respond. UDN pods always have access to KAPI and DNS.

---

## Cleanup

```bash
oc delete -k use-case-udn-services/
```

---

## References

- [Primary networks (UDN) – OpenShift 4.21](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/multiple_networks/primary-networks)
- [Services – Kubernetes](https://kubernetes.io/docs/concepts/services-networking/service/)
