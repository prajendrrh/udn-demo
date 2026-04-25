# 01 — Add OpenShift authentication via LDAP

This topic configures an **LDAP identity provider** for OpenShift authentication (OAuth), so users can log in with their enterprise directory credentials.

## Prerequisites

- OpenShift 4.21 cluster access as `cluster-admin`
- LDAP server reachable from the cluster OAuth pods (network/DNS/firewall)
- LDAP bind DN + password (or anonymous bind if allowed)
- LDAP CA bundle (if using LDAPS / StartTLS with a private CA)

## Procedure

### 1. Confirm current OAuth configuration

```bash
oc get oauth cluster -o yaml
```

### 2. Create the bind password secret

Create a secret in `openshift-config` containing `bindPassword`.

```bash
oc -n openshift-config create secret generic ldap-bind-password \
  --from-literal=bindPassword='<REPLACE_ME>'
```

### 3. (Optional) Create a ConfigMap with the LDAP CA bundle

If your LDAP endpoint uses a private CA, create a ConfigMap with the PEM bundle.

```bash
oc -n openshift-config create configmap ldap-ca \
  --from-file=ca.crt=./ca.crt
```

### 4. Apply the LDAP identity provider configuration

Edit the cluster OAuth resource and add an LDAP identity provider. You can use this file as a starting point.

1. Review and update `oauth-ldap-idp.yaml`.
2. Apply it:

```bash
oc apply -f ./oauth-ldap-idp.yaml
```

### 5. Verify

1. Wait for OAuth to roll out.

```bash
oc -n openshift-authentication get pods
oc -n openshift-authentication rollout status deploy/oauth-openshift
```

2. Test login (example):

```bash
oc login -u <ldap-username> -p '<ldap-password>' https://api.<cluster>.<domain>:6443
```

## Expected output

- `oauth-openshift` deployment successfully rolls out.
- LDAP users can authenticate (login succeeds) and appear under:

```bash
oc get users
oc get identities
```

## References (OpenShift 4.21)

- LDAP identity provider configuration: `https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html-single/authentication_and_authorization/index`
- LDAP group sync (optional follow-up): `http://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/authentication_and_authorization/ldap-syncing`

