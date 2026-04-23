### FILE: execution-plan/execution-plan/11-phase-7-vm-provisioning-placement.md

# Phase 7 — VM Provisioning and Strict Placement

## Objective

Provision the fixed set of application VMs on the designated worker nodes with the approved sizes and node placement rules.

## Entry Criteria

- Phase 6 completed.
- Guest OS media available.
- Worker-3 capacity decision approved.

## Steps

1. Create VM definitions with the fixed vCPU, memory, and disk values.
2. Apply node affinity or equivalent placement controls so each VM runs only on its designated worker.
3. Provision VM disks from the worker-local storage configuration.
4. Build or import the operating systems for each VM.
5. For VM11 ActiveMQ, import the existing service data and configuration according to the approved migration method.
6. Install guest agents, time synchronization, and base OS hardening controls.
7. Record final VM inventory, node placement, IP addressing, and access ownership.

## Strict Placement Map

| Worker | VMs | Aggregate vCPU | Aggregate RAM | Aggregate Disk | Capacity Observation |
|---|---|---|---|---|---|
| Worker-1 | VM3, VM12 | 20 | 112 GB | 750 GB | Fits fixed worker CPU and disk |
| Worker-2 | VM10, VM11 | 20 | 144 GB | 1,000 GB | Fits fixed worker CPU and disk |
| Worker-3 | VM6, VM7, VM8, VM9 | 28 | 208 GB | 1,780 GB | Exceeds fixed worker CPU by 8 vCPU and fixed usable disk by ~480 GB; requires approved client disposition |

## Validation Commands/Tests and Expected Results

| Validation Command/Test | Expected Result |
|---|---|
| `oc get vm,vmi -A -o wide` | All fixed-scope VMs present, and running VMs show intended worker placement |
| `oc get vmi -A -o custom-columns=VM:.metadata.name,NODE:.status.nodeName` | Each VM runs only on its designated worker |
| Guest console or `ssh` login test to each VM | Each VM boots successfully and guest OS access works |
| `oc describe vm <vm-name>` placement-control review | Node affinity or equivalent placement control is present for each VM |
| Capacity validation against the fixed placement table and approved exception log | Worker-3 provisioning proceeds only under approved client decision |

## Exit Criteria

- All fixed-scope VMs provisioned.
- Placement mapping enforced.
- Capacity exception evidence attached for Worker-3 if required.

## Rollback / Exit Plan

- Remove failed or non-compliant VMs, reclaim their local storage, and rebuild from approved templates after correction.
