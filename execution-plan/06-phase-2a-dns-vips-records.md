### FILE: execution-plan/06-phase-2a-dns-vips-records.md

# Phase 2a — DNS Records and VIPs

## Objective
Create the DNS records and VIPs required by a bare-metal OpenShift installation.

## Steps

1. Reserve API VIP and Ingress VIP addresses on the approved cluster network.
   - Official doc: https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/installing_on_bare_metal/installing-bare-metal

2. Create DNS `A` record for `api.<cluster_name>.<base_domain>` pointing to the API VIP.
   - Official doc: https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/installing_on_bare_metal/installing-bare-metal

3. Create DNS `A` record for `api-int.<cluster_name>.<base_domain>` pointing to the API VIP (internal).
   - Official doc: https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/installing_on_bare_metal/installing-bare-metal

4. Create DNS wildcard `A` record for `*.apps.<cluster_name>.<base_domain>` pointing to the Ingress VIP.
   - Official doc: https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/installing_on_bare_metal/installing-bare-metal

5. Create forward and reverse (`PTR`) records for each control plane node, worker node, and the bastion host.
   - Official doc: https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/installing_on_bare_metal/installing-bare-metal

## Validation / Expected Results

- `dig +short api.<cluster>.<baseDomain>` returns the API VIP.
- `dig +short api-int.<cluster>.<baseDomain>` returns the API VIP.
- `dig +short test.apps.<cluster>.<baseDomain>` returns the Ingress VIP.
- `dig -x <node-ip>` returns the expected PTR record for every node.
  - Official doc: https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/installing_on_bare_metal/installing-bare-metal

## Exit Criteria

- All required DNS records verified via forward and reverse lookup.
