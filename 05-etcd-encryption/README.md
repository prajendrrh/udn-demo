# 05 — Enable etcd encryption at rest

This topic enables **etcd encryption** for sensitive resources (like Secrets) stored in etcd.

## Prerequisites

- OpenShift 4.21 cluster access as `cluster-admin`
- Maintenance window (encryption can take time; avoid taking etcd backups mid-migration)

## Procedure

### 1. Check current encryption state

```bash
oc get apiserver cluster -o jsonpath='{.spec.encryption.type}{"\n"}'
oc get openshiftapiserver cluster -o jsonpath='{.spec.encryption.type}{"\n"}'
```

### 2. Enable encryption

Apply `apiserver-encryption-aesgcm.yaml` (recommended default).

```bash
oc apply -f ./apiserver-encryption-aesgcm.yaml
```

### 3. Monitor progress

```bash
oc get apiserver cluster -o yaml | sed -n '1,120p'
oc get openshiftapiserver cluster -o yaml | sed -n '1,160p'
```

Wait until the relevant status conditions indicate encryption completed (per docs).

### 4. Verify resources are being encrypted

Use the documented verification steps and conditions for API server encryption completion.

## Expected output

- `APIServer/cluster` shows `spec.encryption.type: aesgcm`
- Encryption completes (status conditions) without degraded operators

## References (OpenShift 4.21)

- Enabling etcd encryption: `https://docs.redhat.com/es/documentation/openshift_container_platform/4.21/html/etcd/enabling-etcd-encryption`
- Security and compliance overview: `https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html-single/security_and_compliance/`
- etcd backup/restore (do not back up mid-encryption): `http://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/etcd/backing-up-and-restoring-etcd-data`

