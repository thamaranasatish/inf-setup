### FILE: execution-plan/00-index.md

# Execution Plan Index

This folder contains the end-to-end execution plan as a set of phased markdown files. Every technical step is followed by a citation link to an official vendor document. Where no clear official document exists for a step, the step is marked `OFFICIAL DOC NOT FOUND – NEEDS CLIENT CONFIRMATION` and is not executed until the client confirms a supported method.

## Ground Rules

- Architecture and sizing are fixed and not altered by this plan.
- Control plane nodes are dedicated and not schedulable. No VM workloads run on control plane nodes.
- Only official documentation links are cited.

## Files

- Phase 0 — Readiness
  - [04-phase-0a-readiness-prechecks.md](04-phase-0a-readiness-prechecks.md)
- Phase 1 — Hardware
  - [05-phase-1a-bios-firmware-settings.md](05-phase-1a-bios-firmware-settings.md)
  - [05-phase-1b-bmc-ipmi-validation.md](05-phase-1b-bmc-ipmi-validation.md)
- Phase 2 — Network, DNS, Time
  - [06-phase-2a-dns-vips-records.md](06-phase-2a-dns-vips-records.md)
  - [06-phase-2b-network-firewall-prechecks.md](06-phase-2b-network-firewall-prechecks.md)
  - [06-phase-2c-ntp-chrony-setup.md](06-phase-2c-ntp-chrony-setup.md)
- Phase 3 — OpenShift Install
  - [07-phase-3a-openshift-install-baremetal-shell.md](07-phase-3a-openshift-install-baremetal-shell.md)
  - [07-phase-3b-post-install-smoke-tests.md](07-phase-3b-post-install-smoke-tests.md)
- Phase 4 — Security & RBAC
  - [08-phase-4a-rbac-baseline.md](08-phase-4a-rbac-baseline.md)
  - [08-phase-4b-security-hardening-baseline.md](08-phase-4b-security-hardening-baseline.md)
- Phase 5 — OpenShift Virtualization
  - [09-phase-5a-install-openshift-virtualization-operator.md](09-phase-5a-install-openshift-virtualization-operator.md)
  - [09-phase-5b-configure-hyperconverged-cr.md](09-phase-5b-configure-hyperconverged-cr.md)
- Phase 6 — Storage
  - [10-phase-6a-local-ssd-storage-classes-pv-strategy.md](10-phase-6a-local-ssd-storage-classes-pv-strategy.md)
  - [10-phase-6b-rwx-live-migration-limitation-and-options.md](10-phase-6b-rwx-live-migration-limitation-and-options.md)
- Phase 7 — VM Templates and Placement
  - [11-phase-7a-create-vm-templates-and-images.md](11-phase-7a-create-vm-templates-and-images.md)
  - [11-phase-7b-vm-placement-affinity-anti-affinity.md](11-phase-7b-vm-placement-affinity-anti-affinity.md)
- Phase 8 — Application Tooling Install (per VM)
  - [12-phase-8a-vm3-cicd-tools-install.md](12-phase-8a-vm3-cicd-tools-install.md)
  - [12-phase-8b-vm10-sqlserver-install.md](12-phase-8b-vm10-sqlserver-install.md)
  - [12-phase-8c-vm8-vm9-elk-grafana-install.md](12-phase-8c-vm8-vm9-elk-grafana-install.md)
  - [12-phase-8d-vm6-nginx-gateway-install.md](12-phase-8d-vm6-nginx-gateway-install.md)
  - [12-phase-8e-vm7-kafka-cache-install.md](12-phase-8e-vm7-kafka-cache-install.md)
  - [12-phase-8f-vm12-vault-or-cyberark-install.md](12-phase-8f-vm12-vault-or-cyberark-install.md)
- Phase 9 — Validation and Handover
  - [13-phase-9a-end-to-end-validation-scenarios.md](13-phase-9a-end-to-end-validation-scenarios.md)
  - [13-phase-9b-handover-artifacts-and-runbooks.md](13-phase-9b-handover-artifacts-and-runbooks.md)

## Open Client Decisions

- Worker-3 aggregate capacity vs fixed worker size: accept exception, or relocate one of VM7/VM8/VM9 across workers, or increase worker sizing. This plan does not change sizing unilaterally.
- VM10 guest OS (RHEL or Windows Server) for SQL Server 2022.
- VM12 secrets platform (HashiCorp Vault or CyberArk).
- VM7 cache engine (Redis, Red Hat Data Grid / Infinispan, or other).
- Identity provider selection for OCP OAuth (LDAP / OIDC / other).
