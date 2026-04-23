### FILE: execution-plan/execution-plan/03-ownership-raci.md

# 03 Dependency Ownership and RACI

## 3.1 Dependency Ownership Matrix

| Dependency | Description | Owner | Due by Phase | Validation Method |
|---|---|---|---|---|
| Hardware delivery | R750 servers installed, cabled, healthy | Client | Phase 0 | Asset register and hardware health evidence |
| Firmware and BIOS baseline | BIOS, iDRAC, NIC, storage firmware aligned | Platform | Phase 1 | Firmware report and BIOS export |
| BMC access | iDRAC addresses, credentials, remote console validated | Client | Phase 1 | Remote login and power-cycle test |
| Six-node realization method | Supported method for 3 CP + 3 worker on 3 servers approved | Client and Platform | Phase 0 | Signed design decision record |
| OpenShift version selection | Approved OCP release and matching OpenShift Virtualization release | Client and Platform | Phase 0 | Version approval record |
| Content source | Internet egress or client mirror for OCP and Operators | Client and Security | Phase 2 | Registry reachability test |
| Proxy and `noProxy` values | Approved proxy and allow-list for bastion, cluster, and guest traffic | Network and Security | Phase 2 | Bastion connectivity validation |
| VLANs and subnets | Segments, routing, MTU, and gateway details provided | Network | Phase 2 | Approved network sheet |
| DNS records | Node, API, API-int, apps wildcard records created | Network | Phase 2 | Forward and reverse lookup validation |
| VIP delivery method | API and Ingress VIP implementation approved | Network | Phase 2 | VIP design sign-off |
| Firewall approvals | Required cluster and tooling flows opened | Security and Network | Phase 2 | Firewall rule verification |
| NTP services | Approved time sources reachable | Network | Phase 2 | `chronyc` validation |
| Bastion host | Admin host with required tools and access | Platform | Phase 3 | Host readiness checklist |
| Certificates and CA | Cluster and application certificate requirements approved | Security | Phase 4 | Certificate inventory and trust validation |
| AD/LDAP/SSO details | Identity source, groups, mappings approved | Security | Phase 4 | Authentication test plan |
| RBAC model | Admin, operator, app-owner, audit roles approved | Security and Client | Phase 4 | RBAC matrix sign-off |
| Hardening baseline | Approved OS and platform hardening standard | Security and Platform | Phase 4 | Baseline control review |
| PAM and MFA controls | Privileged access, session recording, break-glass workflow | Security | Phase 4 | PAM and MFA access test |
| Vulnerability and EDR onboarding | Scanning and EDR plan approved | Security | Phase 4 | Security tooling readiness review |
| SIEM and evidence retention | Audit-log destination and retention approved | Security | Phase 4 | SIEM onboarding confirmation |
| Local storage layout | Local SSD layout for CP, worker, VM disk use confirmed | Platform | Phase 6 | Storage layout sign-off |
| Worker-3 capacity decision | CPU overcommit and local-disk shortfall disposition approved | Client and Platform | Phase 0 | Signed capacity exception |
| VM guest OS media | Templates or ISO media available | Client | Phase 7 | Media checksum and accessibility validation |
| SQL installation inputs | SQL 2022 edition, licensing, OS, service accounts approved | Client and DBA | Phase 8 | DBA-approved install pack |
| Secrets platform decision | Vault or CyberArk selected for VM12 | Client and Security | Phase 8 | Signed product decision |
| Kafka cache definition | Cache technology on VM7 approved | Client | Phase 8 | Design note approval |
| ELK and Grafana topology | VM8 and VM9 operating pattern approved for DEV | Client and Platform | Phase 8 | Approved service design note |
| Backup scope | DEV backup, export, restore expectations defined | Client and Platform | Phase 9 | Backup scope document |

## 3.2 RACI Summary

Legend: R=Responsible, A=Accountable, C=Consulted, I=Informed.

| Activity | Client | Network | Security | Platform | DBA |
|---|---|---|---|---|---|
| Readiness approval (Phase 0) | A | C | C | R | I |
| BIOS/firmware baseline (Phase 1) | I | I | I | A/R | I |
| Network, DNS, NTP, VIP, firewall (Phase 2) | A | R | C | C | I |
| OpenShift install (Phase 3) | I | C | C | A/R | I |
| Hardening, RBAC, SSO, monitoring (Phase 4) | C | I | A/R | R | I |
| OpenShift Virtualization enable (Phase 5) | I | I | C | A/R | I |
| VM local storage (Phase 6) | C | I | C | A/R | I |
| VM provisioning and placement (Phase 7) | C | I | C | A/R | C |
| Tool installs on VMs (Phase 8) | A | I | C | R | C |
| SQL Server 2022 on VM10 | A | I | C | C | R |
| Validation and handover (Phase 9) | A | C | C | R | C |
