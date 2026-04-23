### FILE: execution-plan/execution-plan/10-phase-6-storage-for-vms.md

# Phase 6 — VM Storage Setup (Local SSD Strategy)

## Objective

Implement the DEV-local storage approach for VM disks using only the local SSD capacity available on each worker node.

## Entry Criteria

- Phase 5 completed.
- Local SSD layout approved.
- Worker-3 capacity decision closed.

## Steps

1. Reserve worker-local disk capacity for OpenShift Virtualization VM disks per the approved local SSD layout.
2. Configure the DEV storage method for node-local VM disks using worker-local storage classes and persistent volumes aligned to worker placement.
3. Define storage classes, reclaim policy, naming standard, and quota controls for VM disks.
4. Document that VM disks are node-local and that shared RWX storage is not present in this scope.
5. Validate image import and test disk provisioning for a sample VM.
6. Record per-worker usable capacity and planned allocations for audit.

## Outputs

- Local VM storage configuration.
- Per-worker capacity ledger.
- Storage limitation statement.

## Validation Commands/Tests and Expected Results

| Validation Command/Test | Expected Result |
|---|---|
| `oc get storageclass` | Approved worker-local storage classes are present |
| `oc get pv,pvc -A | grep <local-storage-class>` | Test PV and PVC objects created successfully |
| `oc describe pvc <test-pvc>` | PVC binds to the intended local storage on the expected worker |
| Capacity review using `oc get pv -A` and the worker allocation ledger | Per-worker allocated disk aligns to the approved placement plan and accepted exception record |
| Storage limitation acknowledgment review | Live-migration limitation for local-only DEV storage is documented and accepted |

## Storage and Availability Note

- DEV uses only the local 2 TB SSD per server for control plane, worker, and VM disks.
- VM disks are node-local. RWX shared storage is not in scope.
- Live migration of VMs across workers is limited or out of scope for DEV because there is no RWX-backed shared storage.
- Worker node failure recovery depends on node restoration, VM rebuild, or the DEV backup or export approach approved by the client.
- PROD would require shared, resilient RWX-capable storage, stronger recovery controls, and headroom for maintenance and failover.

## Exit Criteria

- VM disk storage is available on each worker.
- Storage limitations are formally accepted for DEV.

## Rollback / Exit Plan

- Delete unused storage resources, reclaim unconsumed local capacity, and stop before application VM provisioning if storage validation fails.
