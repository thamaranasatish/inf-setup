### FILE: execution-plan/10-phase-6a-local-ssd-storage-classes-pv-strategy.md

# Phase 6a — Local SSD Storage Classes and PV Strategy

## Objective
Expose local SSD capacity on each worker as persistent storage for VM disks via the Red Hat Local Storage Operator (LSO).

## Steps

1. Install the Local Storage Operator from OperatorHub in the `openshift-local-storage` namespace.
   - Official doc: https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/storage/persistent_storage/persistent-storage-local

2. Create a `LocalVolume` resource that discovers the allocated SSD device(s) on each worker and creates Persistent Volumes with a dedicated `StorageClass` (e.g., `local-vm-disks`).
   - Official doc: https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/storage/persistent_storage/persistent-storage-local

3. Configure OpenShift Virtualization / CDI to use the local `StorageClass` for VM root disks and data disks.
   - Official doc: https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/virtualization/virtual-machine-storage

## Validation / Expected Results

- `oc get storageclass` includes the approved local storage class.
- `oc get pv,pvc -A` shows test PV/PVC objects created on the intended worker.
- `oc describe pvc <test-pvc>` shows the binding worker matches the expected placement.
  - Official doc: https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/storage/persistent_storage/persistent-storage-local

## Exit Criteria

- Local storage class available on each worker and proven to bind test PVCs.
