### FILE: execution-plan/13-phase-9a-end-to-end-validation-scenarios.md

# Phase 9a — End-to-End Validation Scenarios

## Objective
Run an end-to-end validation across the platform and application stack using official doc commands.

## Scenarios

1. **Cluster health**
   ```
   oc get nodes
   oc get co
   oc get clusterversion
   ```
   Expected: 3 CP + 3 worker `Ready`, all operators `Available=True`.
   - Official doc: https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/cli_tools/openshift-cli-oc

2. **Control plane dedication**
   ```
   oc get schedulers.config.openshift.io cluster -o jsonpath='{.spec.mastersSchedulable}'
   ```
   Expected: `false`.
   - Official doc: https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/nodes/working-with-clusters

3. **VM placement**
   ```
   oc get vmi -A -o custom-columns=VM:.metadata.name,NODE:.status.nodeName
   ```
   Expected: Each VM is on its assigned worker per `01-architecture-baseline.md`.
   - Official doc: https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/virtualization/virtual-machines

4. **CI/CD end-to-end**
   - Run a Jenkins pipeline that retrieves a secret from VM12, builds an artifact, publishes to Nexus, and runs SonarQube / Aqua / Checkmarx scans.
   - Official doc (Jenkins Pipeline): https://www.jenkins.io/doc/book/pipeline/
   - Official doc (Nexus): https://help.sonatype.com/en/sonatype-nexus-repository.html
   - Official doc (SonarQube): https://docs.sonarsource.com/sonarqube-server/latest/

5. **Database**
   ```
   sqlcmd -S <sql-host> -Q "SELECT @@VERSION"
   ```
   Expected: SQL Server 2022 version string.
   - Official doc: https://learn.microsoft.com/en-us/sql/tools/sqlcmd/sqlcmd-utility

6. **Messaging**
   - ActiveMQ: test queue produce/consume via official tools.
     - Official doc: https://activemq.apache.org/components/classic/documentation
   - Kafka: `kafka-console-producer.sh` and `kafka-console-consumer.sh`.
     - Official doc: https://kafka.apache.org/documentation/#quickstart

7. **API gateway**
   ```
   curl -sk https://<nginx-host>/health
   ```
   Expected: healthy response per configured route.
   - Official doc: https://docs.nginx.com/nginx/admin-guide/load-balancer/http-health-check/

8. **Observability**
   ```
   curl -s http://<elasticsearch-host>:9200/_cluster/health
   curl -sk https://<grafana-host>/api/health
   ```
   - Official doc: https://www.elastic.co/guide/en/elasticsearch/reference/current/cluster-health.html
   - Official doc: https://grafana.com/docs/grafana/latest/

## Exit Criteria

- All scenarios pass with evidence archived.
