### FILE: execution-plan/execution-plan/00-README.md

# OpenShift Virtualization DEV Execution Plan

This folder contains the enterprise execution plan for the on-prem DEV environment built on three Dell EMC PowerEdge R750 servers running Red Hat OpenShift with OpenShift Virtualization (KubeVirt). The plan is split into dependency-ordered files so delivery teams and client stakeholders can sign off incrementally.

“Red Hat OpenShift Virtualization is a feature included with Red Hat OpenShift that enables organizations to run and manage traditional virtual machines (VMs) alongside containerized workloads on a single, unified Kubernetes platform.”

## Scope Summary

- Three physical servers, each hosting one dedicated OpenShift control plane node and one dedicated worker node.
- Three control plane nodes are dedicated and not schedulable for user workloads.
- Three worker nodes host all application VMs via OpenShift Virtualization.
- No external hypervisor required for application VM execution.
- Local 2 TB SSD per server is the only in-scope storage tier. RWX shared storage is out of scope for DEV.

## Reading Order

| # | File | Purpose |
|---|---|---|
| 1 | `01-architecture-baseline.md` | Fixed architecture, sizing, placement, and simple diagrams |
| 2 | `02-prerequisites-and-dependencies.md` | Pre-execution prerequisites across hardware, network, storage, OS, security, access |
| 3 | `03-ownership-raci.md` | Dependency and ownership matrix and RACI |
| 4 | `04-phase-0-readiness.md` | Phase 0 readiness and validation |
| 5 | `05-phase-1-baremetal-bios-firmware.md` | Phase 1 bare metal, BIOS, firmware |
| 6 | `06-phase-2-network-dns-ntp.md` | Phase 2 network, DNS, NTP, VIPs, firewall |
| 7 | `07-phase-3-openshift-install.md` | Phase 3 OpenShift installation with dedicated control plane |
| 8 | `08-phase-4-postinstall-baseline-hardening.md` | Phase 4 post-install hardening, RBAC, SSO, monitoring |
| 9 | `09-phase-5-openshift-virtualization-enable.md` | Phase 5 OpenShift Virtualization enablement |
| 10 | `10-phase-6-storage-for-vms.md` | Phase 6 VM storage and limitations |
| 11 | `11-phase-7-vm-provisioning-placement.md` | Phase 7 VM provisioning and strict placement |
| 12 | `12-phase-8-vm-software-install-config.md` | Phase 8 tool install and configuration inside VMs |
| 13 | `13-phase-9-validation-handover.md` | Phase 9 validation and handover |
| 14 | `14-runbooks-validation-tests.md` | Consolidated validation commands and expected results |
| 15 | `15-risks-assumptions-decisions.md` | Risks and decisions required from client |
| 16 | `16-go-live-checklist.md` | Go-live and handover readiness checklist |

## Control Plane Statement

Control plane nodes are dedicated, unschedulable for user workloads, and must retain `spec.mastersSchedulable=false` and their default `NoSchedule` taints across the environment lifecycle.
