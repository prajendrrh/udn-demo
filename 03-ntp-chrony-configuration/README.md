# 03 — Configure NTP servers (chrony) via MachineConfig

This topic configures node time synchronization by **overriding `/etc/chrony.conf`** on OpenShift nodes using a `MachineConfig`.

## Prerequisites

- OpenShift 4.21 cluster access as `cluster-admin`
- Reachable NTP server(s) from all nodes (or a local NTP source)
- Maintenance window: applying MachineConfig changes triggers node reboots (one-by-one)

## Procedure

### 1. Review current time sync state (optional)

```bash
oc debug node/<any-node> -- chroot /host bash -lc 'chronyc sources -v || true; chronyc tracking || true'
```

### 2. Update the MachineConfig example

Edit `99-worker-chrony.yaml` and replace the example NTP servers.

### 3. Apply the MachineConfig

```bash
oc apply -f ./99-worker-chrony.yaml
```

### 4. Watch the MachineConfigPool roll out

```bash
oc get mcp
oc wait --for=condition=updated mcp/worker --timeout=60m
```

### 5. Verify chrony is using your servers

```bash
oc debug node/<worker-node> -- chroot /host bash -lc 'grep -n \"^server\\|^pool\" /etc/chrony.conf; chronyc sources -v; chronyc tracking'
```

## Expected output

- Worker `mcp` becomes `UPDATED=True`
- Nodes show your configured `server` entries in `/etc/chrony.conf`
- `chronyc sources -v` shows reachable sources

## References (OpenShift 4.21)

- MachineConfig objects (node configuration): `http://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/machine_configuration/machine-configs-configure`
- Post-install configuration (chrony/NTP is covered as part of post-install topics): `https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/installing_on_bare_metal/bare-metal-post-installation-configuration`

