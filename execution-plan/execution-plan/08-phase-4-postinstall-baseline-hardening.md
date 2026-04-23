### FILE: execution-plan/execution-plan/08-phase-4-postinstall-baseline-hardening.md

# Phase 4 — Post-Install Baseline Hardening, RBAC, and Monitoring

## Objective

Apply the minimum operational and security baseline required before enabling virtualization and onboarding VMs.

## Entry Criteria

- Phase 3 completed.
- Cluster health stable.
- Security and identity inputs approved.

## Steps

1. Configure the approved identity provider integration for AD, LDAP, or SSO.
2. Implement the approved RBAC model for cluster admins, virtualization admins, application owners, and auditors.
3. Configure cluster certificates and trust chain as required for DEV.
4. Enable audit logging, log forwarding, and baseline monitoring integration per client policy.
5. Validate break-glass admin procedure and privileged access custody.
6. Verify control plane nodes remain dedicated and not schedulable after any post-install changes.
7. Record the post-install baseline and export relevant configuration evidence.

## Outputs

- Cluster hardening record.
- Approved RBAC implementation.
- Identity integration evidence.
- Monitoring and audit baseline evidence.

## Validation Commands/Tests and Expected Results

| Validation Command/Test | Expected Result |
|---|---|
| Browser or CLI SSO login test with an approved test identity | Approved identity users authenticate successfully |
| `oc auth can-i` or `oc adm policy who-can` with approved test roles | Only approved groups hold elevated permissions |
| `openssl s_client -connect <api-or-app-endpoint>:443 -showcerts` | Certificate chain matches the approved CA and is within validity |
| Audit log forwarding test to approved SIEM or syslog destination | Audit events are generated and visible at the target destination |
| `oc get co monitoring` and alerting baseline review | Cluster monitoring components are healthy and baseline alerts are present |
| `oc get schedulers.config.openshift.io cluster -o jsonpath='{.spec.mastersSchedulable}'` | Returns `false` |

## Exit Criteria

- Identity, RBAC, and audit baseline operational.
- Cluster meets the agreed DEV hardening baseline.
- Control plane nodes remain dedicated and not schedulable.

## Rollback / Exit Plan

- Remove unapproved identity-provider settings, restore break-glass access, and revert recent policy changes if access or audit functions fail.
