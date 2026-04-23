### FILE: execution-plan/08-phase-4b-security-hardening-baseline.md

# Phase 4b — Security Hardening Baseline

## Objective
Apply the OCP security baseline (audit logs, log forwarding, TLS, pod security).

## Steps

1. Enable and review audit log policies for the API server.
   - Official doc: https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/security_and_compliance/audit-log-view

2. Configure Cluster Logging / Log Forwarder to export audit and infrastructure logs to the approved SIEM destination.
   - Official doc: https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/logging/index

3. Replace default API and Ingress certificates with the approved enterprise CA chain if required.
   - Official doc: https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/security_and_compliance/configuring-certificates

4. Confirm Pod Security admission baseline is `restricted` by default for new user namespaces.
   - Official doc: https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/authentication_and_authorization/understanding-and-managing-pod-security-admission

5. Review and apply Compliance Operator for DEV baseline scanning if the client uses it.
   - Official doc: https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/security_and_compliance/compliance-operator

## Validation / Expected Results

- `openssl s_client -connect api.<cluster>.<baseDomain>:6443 -showcerts` shows the approved CA.
- Audit events reach the approved SIEM/syslog target.
- Default namespace `restricted` PSA enforcement in effect.

## Exit Criteria

- Baseline hardening evidenced, logs forwarding, certificates approved.
