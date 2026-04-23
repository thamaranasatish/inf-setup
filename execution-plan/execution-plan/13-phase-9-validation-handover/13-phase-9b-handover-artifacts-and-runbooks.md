### FILE: execution-plan/13-phase-9b-handover-artifacts-and-runbooks.md

# Phase 9b — Handover Artifacts and Runbooks

## Objective
Assemble and deliver the handover pack required for client sign-off and Day-2 operations.

## Artifacts

- Final execution plan (this folder).
- Logical architecture and network topology diagrams.
- Build inventory of servers, nodes, VMs, IPs, DNS records, VIPs.
- RBAC matrix and access ownership record.
- `kubeconfig` and secrets handled per approved custody; access keys rotated.
- Backup or export policy for DEV.
- Defect list, risk register, and accepted exceptions (including Worker-3 capacity disposition and live-migration limitation).

## Runbooks (Official-Doc-Referenced)

1. OpenShift Day-2 operations (cluster updates, node replacement, etcd backup/restore).
   - Official doc: https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/backup_and_restore/index

2. OpenShift Virtualization VM lifecycle operations (`virtctl` start/stop/pause/migrate within limitations).
   - Official doc: https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/virtualization/virtctl

3. Jenkins administration and backup.
   - Official doc: https://www.jenkins.io/doc/book/system-administration/

4. Nexus Repository backup and upgrade.
   - Official doc: https://help.sonatype.com/en/sonatype-nexus-repository.html

5. SonarQube upgrade and backup.
   - Official doc: https://docs.sonarsource.com/sonarqube-server/latest/setup-and-upgrade/upgrade-the-server/

6. SQL Server 2022 backup and restore.
   - Official doc: https://learn.microsoft.com/en-us/sql/relational-databases/backup-restore/backup-and-restore-of-sql-server-databases

7. ActiveMQ administration.
   - Official doc: https://activemq.apache.org/components/classic/documentation

8. Kafka administration.
   - Official doc: https://kafka.apache.org/documentation/#operations

9. Elastic Stack operations.
   - Official doc: https://www.elastic.co/guide/en/elasticsearch/reference/current/setup.html

10. Grafana administration.
    - Official doc: https://grafana.com/docs/grafana/latest/administration/

11. F5 NGINX administration.
    - Official doc: https://docs.nginx.com/nginx/admin-guide/

12. HashiCorp Vault administration or CyberArk administration (per selection).
    - Official doc (Vault): https://developer.hashicorp.com/vault/docs
    - Official doc (CyberArk): https://docs.cyberark.com/

## Exit Criteria

- Signed handover record or residual-risk acceptance completed; DEV environment formally handed over.
