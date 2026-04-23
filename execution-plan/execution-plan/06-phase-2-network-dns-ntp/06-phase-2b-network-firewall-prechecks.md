### FILE: execution-plan/06-phase-2b-network-firewall-prechecks.md

# Phase 2b — Network and Firewall Pre-Checks

## Objective
Validate required network paths and firewall rules for a bare-metal OpenShift install.

## Steps

1. Open and verify the required cluster ports (control plane, worker, Ingress) per the official OCP networking requirements.
   - Official doc: https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/installing_on_bare_metal/installing-bare-metal
   - Official doc: https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/networking/index

2. Confirm outbound connectivity (direct or via proxy) to the Red Hat registries, or to the approved mirror registry if disconnected.
   - Official doc: https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/installing/about-installing#installation-about-mirror-registry_about-installing
   - Official doc: https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/disconnected_installation_mirroring/index

3. Document and apply proxy / `noProxy` values in `install-config.yaml` if the environment requires an outbound proxy.
   - Official doc: https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/installing/installing-customizations#installation-configure-proxy_installing-customizations

4. Verify identity provider endpoints (LDAP/OIDC) reachability from the bastion and the cluster networks.
   - Official doc: https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/authentication_and_authorization/index

## Validation / Expected Results

- `nc -vz <registry-or-mirror> 443` succeeds from bastion.
- `nc -vz <idp-endpoint> 443` succeeds from bastion.
- Firewall rule test matrix signed off by network and security.

## Exit Criteria

- All cluster-required ports and egress flows evidenced as open before Phase 3.
