### FILE: execution-plan/08-phase-4a-rbac-baseline.md

# Phase 4a — RBAC and Identity Baseline

## Objective
Integrate identity and apply the approved RBAC model.

## Steps

1. Configure an identity provider on the OAuth resource (LDAP, OIDC, or HTPasswd per the client decision).
   - Official doc: https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/authentication_and_authorization/understanding-identity-provider

2. For LDAP integration, use the LDAP identity provider fields (`url`, `bindDN`, `bindPassword`, `ca`, `attributes`) as specified in the OCP docs.
   - Official doc: https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/authentication_and_authorization/configuring-ldap-identity-provider

3. Map approved groups to the following roles: `cluster-admin`, virtualization admins, application admins, read-only, audit.
   - Official doc: https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/authentication_and_authorization/using-rbac

4. Remove the default `kubeadmin` user after a break-glass account has been validated.
   - Official doc: https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/authentication_and_authorization/removing-kubeadmin-user

## Validation / Expected Results

- Login via approved identity provider succeeds.
- `oc auth can-i '*' '*' --as <test-admin-group>` returns `yes` only for the approved admin group.
- Least-privilege verified for a non-admin test user.
  - Official doc: https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/authentication_and_authorization/using-rbac

## Exit Criteria

- Identity integrated, RBAC matrix implemented, `kubeadmin` removed.
