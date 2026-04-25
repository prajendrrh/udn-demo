# PoC — OCP Day 2 Operations

This repository is a structured proof of concept (PoC) for **OpenShift Day 2 operations**. Each folder is a self-contained, step-by-step mini-runbook with the minimal manifests and `oc` commands needed to demonstrate an operational task end-to-end.

## Environment

- OpenShift: (fill in once chosen)
- Access: cluster-admin (recommended for most Day 2 tasks)
- CLI tools: `oc`, `kubectl` (optional), `jq` (optional)

## Repo map (topics)

Each topic is a numbered folder:

- `01-ldap-authentication/`: Add OpenShift authentication via LDAP
- `02-infra-nodes-and-monitoring-placement/`: Label infra nodes and move platform monitoring workloads
- `03-ntp-chrony-configuration/`: Configure NTP servers (chrony) via MachineConfig
- `04-log-forwarding-to-siem/`: Forward logs to a local SIEM system
- `05-etcd-encryption/`: Enable etcd encryption at rest
- `06-logging-and-monitoring-setup/`: Logging and monitoring setup (OpenShift 4.21 docs)

Each folder must contain:

- `README.md` with prerequisites, numbered procedure, and expected output
- Minimal YAML manifests (only when required), stored alongside the README (no `manifests/` subfolder)

## Quick start

1. Pick the topic folder you want to run first.
2. Follow that folder’s `README.md` from top to bottom.

## Conventions

- All docs are in English.
- Prefer declarative resources (YAML) over manual imperative steps when possible.
- Include official documentation links for every operator/API/feature used: `https://docs.redhat.com` or `https://docs.openshift.com`.

