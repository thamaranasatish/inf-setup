### FILE: execution-plan/12-phase-8a-vm3-cicd-tools-install.md

# Phase 8a — VM3 CI/CD Tools Install

## Objective
Install and configure the CI/CD stack inside VM3 per official vendor docs.

## Steps

1. Install and start Jenkins LTS on RHEL per the official installation guide, configure as a systemd service, and set initial admin credentials.
   - Official doc: https://www.jenkins.io/doc/book/installing/linux/
   - Official doc: https://www.jenkins.io/doc/book/system-administration/

2. Install Sonatype Nexus Repository 3 (OSS/PRO) as per the official install-as-a-service Linux guide.
   - Official doc: https://help.sonatype.com/en/installation-and-upgrades.html
   - Official doc: https://help.sonatype.com/en/install-as-a-service-on-linux.html

3. Install SonarQube Server per official guide and configure quality profiles and gates.
   - Official doc: https://docs.sonarsource.com/sonarqube-server/latest/setup-and-upgrade/install-the-server/installing-the-server/
   - Official doc: https://docs.sonarsource.com/sonarqube-server/latest/

4. Install Aqua Enterprise / Aqua components per Aqua official docs (deployment model per client approval).
   - Official doc: https://docs.aquasec.com/

5. Install Checkmarx per official documentation (product and edition per client approval).
   - Official doc: https://docs.checkmarx.com/

6. Install Redgate tools on VM3 per official documentation.
   - Official doc: https://documentation.red-gate.com/

7. Install build tooling: Maven, OpenJDK, Node.js, Newman.
   - Official doc (Maven): https://maven.apache.org/install.html
   - Official doc (OpenJDK / RHEL): https://docs.redhat.com/en/documentation/red_hat_build_of_openjdk
   - Official doc (Node.js): https://nodejs.org/en/download
   - Official doc (Newman): https://learning.postman.com/docs/collections/using-newman-cli/newman-cli-overview/

8. Integrate Jenkins with Vault or CyberArk (per VM12 selection) to retrieve secrets at pipeline runtime.
   - Official doc (Vault): https://developer.hashicorp.com/vault/docs/platform/k8s
   - Official doc (CyberArk Conjur Jenkins plugin): https://docs.cyberark.com/conjur-enterprise/latest/en/content/integrations/cjr_jenkins_int.htm

## Validation / Expected Results

- `systemctl status jenkins nexus sonarqube` on VM3 reports `active (running)`.
- Jenkins UI reachable over HTTPS per enterprise TLS policy.
- Nexus UI reachable and Maven/npm proxies configured.
- SonarQube UI reachable and quality gate defined.
- Smoke pipeline retrieves a secret from VM12 and publishes an artifact to Nexus.

## Exit Criteria

- VM3 CI/CD stack installed and smoke-tested end-to-end.
