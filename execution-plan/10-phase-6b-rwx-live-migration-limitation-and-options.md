### FILE: execution-plan/10-phase-6b-rwx-live-migration-limitation-and-options.md

# Phase 6b — RWX / Live Migration Limitation and Options

## Objective
Document the live-migration limitation under local-only DEV storage and record the future options.

## Statement of Limitation

- Live migration of VMs in OpenShift Virtualization requires the VM’s disks to be backed by `ReadWriteMany` (RWX) storage.
- In this DEV scope, VM disks are provisioned on worker-local SSD via the Local Storage Operator, which exposes `ReadWriteOnce` (RWO) local PVs. This means live migration is not supported for these VMs in DEV.
   - Official doc: https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/virtualization/live-migration
   - Official doc: https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/storage/persistent_storage/persistent-storage-local

## Future Options (For PROD / Later Scope)

- Use Red Hat OpenShift Data Foundation (ODF) to provide RWX-capable shared storage for VM disks.
   - Official doc: https://docs.redhat.com/en/documentation/red_hat_openshift_data_foundation
- Use any Red Hat-supported CSI driver that provides RWX-capable volumes suitable for OpenShift Virtualization live migration.
   - Official doc: https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/storage/using-container-storage-interface-csi

## Validation / Expected Results

- Storage limitation documented and accepted by client in the risk register.
- Future RWX option decided or recorded as out-of-scope for DEV.

## Exit Criteria

- Limitation acknowledged; DEV recovery approach (node repair / rebuild / restore) agreed.
