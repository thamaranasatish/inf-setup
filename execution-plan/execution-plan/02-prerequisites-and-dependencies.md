### FILE: execution-plan/execution-plan/02-prerequisites-and-dependencies.md

# 02 Pre-Execution Prerequisites and Dependencies

Execution is contingent upon the client providing all prerequisite hardware readiness, network services, storage layout confirmation, security integration inputs, access approvals, content-source decisions, and installation-method approvals before delivery execution starts. No phase may begin unless its entry criteria are satisfied and recorded.

## 2.1 Hardware Prerequisites

- Three Dell EMC PowerEdge R750 servers racked, powered, cabled, and inventoried with serial number, asset tag, MAC address, and iDRAC endpoint recorded.
- BIOS, iDRAC, RAID or HBA controller, NIC, and storage firmware updated to a client-approved and Red Hat-compatible baseline.
- UEFI boot enabled consistently on all servers.
- Intel VT-x and VT-d enabled on all servers. Hyper-threading or SMT configured consistently across the server set.
- Remote BMC or iDRAC access, virtual media, power-cycle control, and remote console access available to the platform delivery team.
- Hardware health shows no critical disk, memory, power, or thermal faults.
- Client must confirm the supported method used to realize one control plane node and one worker node per physical server without silently introducing an unapproved hypervisor dependency.

## 2.2 Network Prerequisites

- Client-approved VLANs, subnets, gateways, MTU, routing domains, and switch port configuration for all OpenShift node networks and supporting services.
- API VIP and Ingress VIP addresses reserved before installation.
- If outbound proxying is required, client must provide proxy URLs, `noProxy` values, and the approved egress allow-list for OpenShift, Operator, guest OS, and tool installation traffic.
- Required DNS records created before Phase 3: `api`, `api-int`, and `*.apps` for the selected cluster name and base domain.
- Forward and reverse DNS records for every control plane node, worker node, bastion host, and supporting install endpoint.
- Enterprise NTP reachable from every node and bastion host.
- VIP health-check, failover, and monitoring method for the client-approved VIP delivery mechanism defined before Phase 2 closes.
- Firewall paths approved for node-to-node, bastion-to-cluster admin, outbound content, identity, and workload-to-enterprise-service traffic.
- AD, LDAP, or SSO endpoints reachable from cluster and validated by security team.

## 2.3 Storage Prerequisites

- The only in-scope storage is the local 2 TB SSD per physical server.
- Client confirms the SSD presentation model, RAID mode, HBA or JBOD mode, and disk layout used to satisfy one 300 GB control plane node and one worker node with approximately 1.3 TB usable disk per server.
- Disk layout preserves overhead for installation, filesystem operation, and node maintenance.
- Local SSD reserved exclusively for OpenShift Virtualization storage; no unrelated host consumption.
- RWX shared storage is not in scope for this DEV execution unless explicitly added.
- Worker-3 fixed VM placement requires 1,780 GB of VM disk allocation against a fixed usable disk of approximately 1.3 TB. This is a blocking capacity dependency.

## 2.4 OS Prerequisites

- Client-approved OpenShift release and matching OpenShift Virtualization-supported release selected before Phase 3.
- Supported RHCOS-based installation workflow approved for the six target nodes.
- Bastion or jump host available. If not provided, a RHEL 8 or RHEL 9 bastion is required to host `oc`, `virtctl`, `openshift-install`, SSH keys, and installation artifacts.
- Red Hat pull secret, subscription entitlements, and required registry or mirror access available.
- SSH public keys, bastion hardening baseline, endpoint protection requirements, and approved administrative tooling for the jump host provided before execution.
- For disconnected or proxied environments, client provides approved mirror-registry design, image source list, and bastion-level proxy configuration before Phase 3.
- Guest OS templates or installation media for each VM provided or approved before Phase 7.

## 2.5 Security Prerequisites

- TLS certificates, wildcard application certificate requirements, and the internal CA trust chain supplied or approved.
- AD, LDAP, or SSO integration requirements, group mappings, and identity source endpoints approved by security.
- Break-glass admin accounts, service accounts, SSH key ownership, and credential custody processes defined.
- Target RBAC model for cluster admin, virtualization admin, application admin, read-only, and audit access approved.
- Security scanning endpoints, proxy rules, license servers, and update paths for Aqua, Checkmarx, SonarQube, and related tools reachable.
- Audit log forwarding targets, SIEM or syslog destinations, and retention requirements for DEV defined.
- Enterprise hardening baseline for RHCOS, bastion, Linux guest VMs, and any Windows guest VMs provided or formally waived.
- MFA, privileged access management, session recording, and password vaulting requirements defined.
- Vulnerability scanning scope, scan windows, remediation SLAs, and exception-management process for cluster, bastion, and guest VMs approved.
- Endpoint detection and response, anti-malware, or host-based security agent requirements for guest VMs supplied with required exclusions for OpenShift and application tooling before Phase 8.
- Compliance classification for DEV, evidence retention, log retention, and CMDB or asset registration requirements defined.
- Certificate renewal ownership, service-account rotation cadence, encryption standards, and DEV backup encryption expectations defined.
- Client selects HashiCorp Vault or CyberArk for VM12 before Phase 8.

## 2.6 Access Prerequisites

- Named delivery resources have approved access to iDRAC, switch, firewall, DNS, AD/LDAP, and bastion teams.
- SSH, console, and privileged-escalation paths for all guest OSes approved.
- Change and maintenance windows, rollback authority, and client approvers named before any infrastructure changes.
- Access to source control and any internal package repositories or license portals required during tool installation available.
- Named service or integration accounts for identity, logging, monitoring, backup, and package-repository access created or declared out of scope.
- VPN, jump-host allow-listing, MFA enforcement, and emergency-access approval paths approved before Phase 1.

## 2.7 Hypervisor and Virtualization Confirmation

- OpenShift Virtualization is the default and preferred virtualization layer for application VMs.
- No external hypervisor required for application VM execution.
- No VMware, ESXi, or Hyper-V is assumed.
- If the client elects any host-level virtualization layer to satisfy the six-node layout, it must be raised, justified, and approved as a formal exception before Phase 3.

## 2.8 Network and Access Checklist

| Item | Requirement | Owner | Required By |
|---|---|---|---|
| Cluster base domain | Client-approved cluster name and base domain | Client | Phase 2 |
| API DNS | `api.<cluster>.<baseDomain>` resolves to API VIP | Network | Phase 2 |
| Internal API DNS | `api-int.<cluster>.<baseDomain>` resolves to API VIP | Network | Phase 2 |
| Apps wildcard DNS | `*.apps.<cluster>.<baseDomain>` resolves to Ingress VIP | Network | Phase 2 |
| Node DNS | A and PTR records for 3 control plane nodes, 3 worker nodes, bastion, and supporting endpoints | Network | Phase 2 |
| VIP allocation | One API VIP and one Ingress VIP reserved and routable | Network | Phase 2 |
| Proxy and `noProxy` | If required, approved proxy settings and allow-list | Network and Security | Phase 2 |
| Content source reachability | Red Hat registry or approved internal mirror reachable | Client and Security | Phase 2 |
| Firewall readiness | Admin, identity, NTP, DNS, registry, workload flows approved | Security and Network | Phase 2 |
| NTP | All nodes and bastion sync to approved NTP sources | Network | Phase 2 |
| AD/LDAP/SSO | Identity source reachable with approved bind or trust details | Security | Phase 4 |
| RBAC readiness | Admin, virtualization admin, app owner, and audit roles approved | Security and Client | Phase 4 |
| PAM and MFA | Privileged-access path for bastion, cluster, guest VMs approved | Security | Phase 4 |
| SIEM and retention | Audit-log destination and retention approved | Security | Phase 4 |
