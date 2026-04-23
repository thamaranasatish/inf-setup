### FILE: execution-plan/execution-plan/15-risks-assumptions-decisions.md

# 15 Risks, Assumptions, and Decisions Required

## 15.1 Risks

- The fixed requirement to run six OpenShift nodes on three physical servers cannot be executed until the client confirms the supported implementation method for that layout.
- Worker-3 fixed placement exceeds stated worker capacity by 8 vCPU and approximately 480 GB of usable local disk, creating a direct delivery risk if unresolved before provisioning.
- Local-only SSD storage limits VM mobility and increases recovery dependency on rebuild, restore, or node repair.
- Combined tooling on VM3 can create operational contention in DEV if pipeline usage, scanning jobs, and artifact activity overlap.
- VM12 is a single DEV secrets VM; no application-level HA is in scope.
- VM11 is an existing ActiveMQ workload and may carry undocumented configuration or migration dependencies.

## 15.2 Assumptions

No unresolved planning assumptions are retained in this plan. Items that require client confirmation or approval have been moved to **Decisions Required from Client**.

## 15.3 Decisions Required from Client

- Select the target OpenShift version and confirm the matching OpenShift Virtualization-supported release.
- Confirm the supported method used to instantiate one control plane node and one worker node per physical server.
- Approve the disposition for the Worker-3 CPU and disk shortfall against the fixed placement and fixed worker size.
- Confirm the on-prem API VIP and Ingress VIP delivery method.
- Confirm proxy, `noProxy`, and outbound allow-list requirements for bastion, cluster, Operators, guest OS patching, and tool installation paths.
- Confirm whether the environment is connected to Red Hat registries or requires a client-provided mirror.
- Confirm that all required network, identity, firewall, certificate, logging, monitoring, and content-source dependencies will be delivered before the relevant phase gates.
- Confirm that DEV acceptance criteria allow local-only VM storage without RWX-backed live migration.
- Confirm the approved security-hardening baseline, vulnerability-management process, EDR or anti-malware requirements, and SIEM onboarding expectations.
- Confirm the approved privileged-access model including MFA, PAM, session recording, and break-glass workflow.
- Confirm that the client will provide the required licenses, binaries, install media, templates, and subscriptions for the application stack before Phases 7 and 8.
- Select HashiCorp Vault or CyberArk for VM12.
- Define the cache technology paired with Kafka on VM7.
- Define the intended ELK and Grafana operating pattern across VM8 and VM9 for DEV.
- Provide the base domain, cluster name, VLANs, subnets, gateways, DNS records, and NTP sources.
- Provide certificate requirements, CA chain, and identity integration details.
- Define guest OS standards and install media for each VM, including the OS for SQL Server 2022.
- Confirm SQL Server 2022 edition, licensing model, service accounts, and backup expectations.
- Confirm DEV backup, export, and restore expectations for node-local VM disks and application configurations.
