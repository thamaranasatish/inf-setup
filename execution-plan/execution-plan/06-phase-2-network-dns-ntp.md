### FILE: execution-plan/execution-plan/06-phase-2-network-dns-ntp.md

# Phase 2 — Network, DNS, and NTP Readiness

## Objective

Establish all required name resolution, VIP, time, routing, and firewall dependencies before OpenShift installation.

## Entry Criteria

- Phase 1 completed.
- Network design inputs approved.
- Firewall change process active.

## Steps

1. Reserve IP addresses for all control plane nodes, worker nodes, bastion host, API VIP, and Ingress VIP.
2. Create forward and reverse DNS records for nodes and cluster endpoints (`api`, `api-int`, `*.apps`).
3. Implement the client-approved API VIP and Ingress VIP delivery method.
4. Validate L3 routing from bastion, nodes, and supporting services.
5. Open and verify required firewall paths for install, administration, identity, content source, and application flows.
6. Validate enterprise NTP reachability and time synchronization from the bastion host and representative node networks.
7. Validate identity-source reachability if AD, LDAP, or SSO will be integrated post-install.

## Outputs

- Network readiness sheet.
- DNS and VIP evidence.
- Firewall approval record.
- NTP validation evidence.

## Validation Commands/Tests and Expected Results

| Validation Command/Test | Expected Result |
|---|---|
| `nslookup api.<cluster>.<baseDomain>` and `nslookup api-int.<cluster>.<baseDomain>` from bastion | Both names resolve to the approved API VIP |
| `nslookup test.apps.<cluster>.<baseDomain>` from bastion | Test wildcard application name resolves to the approved Ingress VIP |
| `chronyc sources -v` from bastion | Approved NTP sources are reachable and selected |
| `nc -vz <mirror-or-registry-endpoint> 443` and `nc -vz <idp-endpoint> 443` from bastion | Required outbound HTTPS paths are open |
| Firewall rule test matrix executed by network and security teams | Approved ports and routes succeed between required endpoints |

## Exit Criteria

- All cluster DNS and VIP records exist.
- Firewall and NTP dependencies confirmed.

## Rollback / Exit Plan

- Revert unapproved network changes and pause execution if DNS, VIP, or firewall gaps remain.
