### FILE: execution-plan/06-phase-2c-ntp-chrony-setup.md

# Phase 2c — NTP / chrony Setup

## Objective
Ensure all nodes synchronize to approved enterprise time sources as required by OpenShift.

## Steps

1. Configure the cluster to use approved NTP servers. On RHCOS, NTP is managed by chrony and is set via a MachineConfig at install or Day-2.
   - Official doc: https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/installing_on_bare_metal/installing-bare-metal
   - Official doc: https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/machine_configuration/index

2. Configure chrony on the RHEL bastion host to sync with the same NTP sources.
   - Official doc: https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/configuring_basic_system_settings/assembly_configuring-time-synchronization_configuring-basic-system-settings
   - Official doc: https://chrony-project.org/documentation.html

## Validation / Expected Results

- `chronyc sources -v` on the bastion shows approved NTP sources selected (`^*`).
- `chronyc tracking` reports a stable reference and small estimated error.
  - Official doc: https://chrony-project.org/documentation.html

## Exit Criteria

- Bastion and all planned cluster nodes sync to approved NTP sources.
