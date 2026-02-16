# Use Case 4: Pods on UDN + NetworkPolicy

Pods run on a **UserDefinedNetwork** (`100.3.0.0/24`) with a **NetworkPolicy** that allows ingress only from the same namespace. This demonstrates UDN plus policy-based isolation.

**Optional:** If OpenShift Virtualization is installed, you can also run VMs in this namespace; they will get IPs from the same UDN. See [Optional: Using this UDN for VMs](#optional-using-this-udn-for-vms) at the end.

## Apply

```bash
oc apply -k .
```

Wait for pods:

```bash
oc get pods -n udn-vm-demo
```

## Show UDN IPs

`oc get pods -o wide` shows the default cluster IP. To list each pod and its UDN IP from the annotation (requires `jq`):

```bash
echo "=== udn-vm-demo (UDN 100.3.0.0/24) ==="
oc get pods -n udn-vm-demo -l app=app-udn-vm -o json | jq -r '.items[] | .metadata.name + ": " + ((.metadata.annotations["k8s.v1.cni.cncf.io/network-status"] // "[]") | fromjson | map(select(.ips[0] | startswith("100.3.0."))) | .[0].ips[0] // "?")'
```

## Test connectivity

Use UDN IPs from **Show UDN IPs**. Checks use TCP (bash `/dev/tcp`), no `ping`.

**1. Same namespace (allowed by NetworkPolicy)**  
From one pod, connect to another pod’s UDN IP in the same namespace. You should see "Connection refused" (reachable).

```bash
POD1=$(oc get pod -n udn-vm-demo -l app=app-udn-vm -o jsonpath='{.items[0].metadata.name}')
IP2=$(oc get pods -n udn-vm-demo -l app=app-udn-vm -o json | jq -r '.items[1].metadata.annotations["k8s.v1.cni.cncf.io/network-status"] | fromjson | map(select(.ips[0] | startswith("100.3.0."))) | .[0].ips[0]')
echo "From $POD1 to same-namespace UDN IP $IP2:"
oc exec -n udn-vm-demo $POD1 -- bash -c "timeout 2 bash -c 'echo >/dev/tcp/'"$IP2"'/80' 2>&1; echo exit: \$?"
```
Expected: `Connection refused` and exit 1.

**2. NetworkPolicy: ingress from another namespace blocked**  
From a pod in another namespace (e.g. `default`), try TCP to a `udn-vm-demo` pod’s UDN IP. The policy allows only same-namespace ingress, so you should see timeout.

```bash
# Create a temporary pod in default, then try to reach a UDN IP (replace with actual from Show UDN IPs)
oc run test-policy --image=quay.io/rh_ee_prajendr/ubi-minimal-nettools:latest --restart=Never -n default -- sleep 60
sleep 10
IP_UDN=$(oc get pods -n udn-vm-demo -l app=app-udn-vm -o json | jq -r '.items[0].metadata.annotations["k8s.v1.cni.cncf.io/network-status"] | fromjson | map(select(.ips[0] | startswith("100.3.0."))) | .[0].ips[0]')
oc exec -n default test-policy -- bash -c "timeout 2 bash -c 'echo >/dev/tcp/'"$IP_UDN"'/80' 2>&1; echo exit: \$?"
# Expected: timeout (exit 124). Clean up: oc delete pod test-policy -n default
```
Expected: timeout — policy blocks ingress from other namespaces. Delete the test pod when done: `oc delete pod test-policy -n default`.

## Cleanup

```bash
oc delete -k .
```

## Optional: Using this UDN for VMs

If **OpenShift Virtualization** is installed, you can create VMs in the same namespace `udn-vm-demo`. They will use this UDN and get IPs from `100.3.0.0/24` (Persistent IPAM). Create VMs via the OpenShift console or `VirtualMachine` manifests; no extra network config is required. The same NetworkPolicy applies: only ingress from the same namespace is allowed.
