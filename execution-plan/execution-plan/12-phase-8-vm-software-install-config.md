### FILE: execution-plan/execution-plan/12-phase-8-vm-software-install-config.md

# Phase 8 — Tool Installation and Configuration Inside VMs

## Objective

Install and configure the required application stack components inside the provisioned VMs for the DEV environment.

## Entry Criteria

- Phase 7 completed.
- Guest OS available on each VM.
- Product decisions, licenses, media, service accounts, and secrets-platform selection approved.

## Steps

1. Harden each guest OS, apply required patch levels, and configure time sync, DNS, and approved access controls.
2. **VM3 CI/CD**: install and configure Jenkins, Nexus, SonarQube, Aqua, Checkmarx, Redgate, Maven, JDK, Newman, and NodeJS.
3. **VM12 Secrets**: install and configure the client-selected secrets platform (HashiCorp Vault OR CyberArk).
4. **VM10 SQL Server 2022**: install using the client-approved edition, OS, service accounts, and storage layout.
5. **VM11 ActiveMQ**: restore or reconfigure the existing ActiveMQ workload via the approved migration procedure.
6. **VM6 NGINX**: install F5 NGINX API Gateway and import or create required policies and certificates.
7. **VM7 Kafka + Cache**: install Kafka and the client-specified cache.
8. **VM8 and VM9 ELK + Grafana**: implement the client-approved DEV topology and ingest paths.
9. Integrate VM3 with VM12 so CI/CD jobs obtain secrets from the approved secrets platform (no cleartext credentials on VM3).
10. Configure tool-level logging, backups or exports as defined for DEV, and role-based access controls.

## Outputs

- Configured application VMs.
- Tool access records.
- Secrets integration evidence.
- Service configuration baseline.

## Validation Commands/Tests and Expected Results

| Validation Command/Test | Expected Result |
|---|---|
| `systemctl status` for Jenkins, Nexus, SonarQube on VM3 | Core CI/CD services are active |
| CI/CD smoke pipeline run on VM3 | Jenkins executes a pipeline using the approved toolchain successfully |
| Secrets retrieval test from Jenkins to VM12 via approved plugin/API | Secrets retrieved successfully without cleartext credential storage |
| `sqlcmd -S <sql-host> -Q "SELECT @@VERSION"` | SQL Server 2022 responds successfully |
| Queue or topic smoke test for ActiveMQ and Kafka | Test message send and receive operations succeed |
| `curl -sk https://<nginx-host>/health` | API gateway returns expected healthy response |
| `curl -s http://<elasticsearch-host>:9200/_cluster/health` and `curl -sk https://<grafana-host>/api/health` | ELK and Grafana endpoints respond healthy for DEV |

## Exit Criteria

- All required tools installed and reachable.
- Secrets, logging, and access controls functional.
- Tool-level smoke tests passed.

## Rollback / Exit Plan

- Revert failed service configuration where possible, or rebuild the affected VM from the approved baseline image and reapply validated configuration.
