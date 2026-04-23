### FILE: execution-plan/11-phase-7a-create-vm-templates-and-images.md

# Phase 7a — Create VM Templates and Images

## Objective
Create approved VM templates and import base images for the required guest OS families.

## Steps

1. For RHEL-based guests, import a RHEL cloud image or use a pre-approved template from the OpenShift Virtualization templates catalog.
   - Official doc: https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/virtualization/virtual-machines#virt-creating-vms-from-templates
   - Official doc: https://access.redhat.com/downloads/content/rhel

2. For Windows-based guests (if required for VM10), follow the official Microsoft SQL Server 2022 on Windows Server guidance and the OpenShift Virtualization Windows template guidance. If a Linux OS is selected for VM10, follow §12 `12-phase-8b`.
   - Official doc: https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/virtualization/virtual-machines

3. Use the Containerized Data Importer (CDI) DataVolume / DataSource to populate VM disks from approved OS images.
   - Official doc: https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/virtualization/virtual-machine-storage

4. Record approved CPU, memory, and disk defaults in each template matching this plan’s fixed sizing.
   - Official doc: https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/virtualization/virtual-machines

## Validation / Expected Results

- Templates visible under **Virtualization → Templates** in the console.
- CDI image import succeeds:
  ```
  oc get datavolume -A
  ```
  - Official doc: https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/virtualization/virtual-machine-storage

## Exit Criteria

- Approved templates and base images ready for the fixed VM set.
