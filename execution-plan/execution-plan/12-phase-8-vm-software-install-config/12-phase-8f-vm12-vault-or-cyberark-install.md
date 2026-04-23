### FILE: execution-plan/12-phase-8f-vm12-vault-or-cyberark-install.md

# Phase 8f — VM12 HashiCorp Vault OR CyberArk Install

## Objective
Install the client-selected secrets platform on VM12 per official vendor docs.

> The client must confirm Vault or CyberArk before execution. The procedure for the unselected product is not executed.

## Option A — HashiCorp Vault

1. Install Vault on RHEL per official installation guide (RPM repo or binary).
   - Official doc: https://developer.hashicorp.com/vault/install
   - Official doc: https://developer.hashicorp.com/vault/tutorials/operations/deployment-guide

2. Configure the storage backend (integrated Raft or supported backend) per deployment guide.
   - Official doc: https://developer.hashicorp.com/vault/docs/configuration/storage/raft

3. Initialize and unseal Vault; store recovery / unseal keys under client-approved custody.
   - Official doc: https://developer.hashicorp.com/vault/tutorials/getting-started/getting-started-deploy
   - Official doc: https://developer.hashicorp.com/vault/docs/commands/operator/init

4. Enable secrets engines and authentication methods required for Jenkins integration.
   - Official doc: https://developer.hashicorp.com/vault/docs/secrets
   - Official doc: https://developer.hashicorp.com/vault/docs/auth

5. Integrate Jenkins with Vault using the official HashiCorp guidance or Jenkins plugin.
   - Official doc: https://developer.hashicorp.com/vault/docs/platform/k8s

## Option B — CyberArk

1. Install the chosen CyberArk product (Self-Hosted / PAM, Conjur Enterprise, or Conjur OSS) per official CyberArk documentation.
   - Official doc: https://docs.cyberark.com/

2. For Jenkins integration with CyberArk Conjur, follow the official Jenkins integration guide.
   - Official doc: https://docs.cyberark.com/conjur-enterprise/latest/en/content/integrations/cjr_jenkins_int.htm

## Validation / Expected Results

- Vault: `vault status` returns `Sealed: false` after init/unseal, and a test secret can be written and read with an approved token or auth method.
- CyberArk: a test credential retrieval from Jenkins to CyberArk Conjur returns the expected secret value.

## Exit Criteria

- VM12 operational and integrated with VM3 CI/CD.
