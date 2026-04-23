### FILE: execution-plan/execution-plan/05-phase-1-baremetal-bios-firmware.md

# Phase 1 — Bare Metal and Firmware/BIOS Preparation

## Objective

Prepare all three Dell EMC PowerEdge R750 servers to a common, supportable baseline for OpenShift installation.

## Entry Criteria

- Phase 0 exit criteria met.
- Maintenance window approved.
- iDRAC access and firmware packages available.

## Steps

1. Validate hardware health and capture current firmware and BIOS levels.
2. Apply approved BIOS, iDRAC, NIC, and storage-controller firmware updates.
3. Set consistent UEFI boot configuration across all servers.
4. Enable Intel VT-x and VT-d on all servers.
5. Apply the client-approved local SSD presentation model and record the resulting disk layout.
6. Validate remote console, virtual media, and power control for each server.
7. Record final MAC addresses, NIC mappings, and storage presentation details for install use.

## Outputs

- Firmware baseline report.
- BIOS configuration evidence.
- Hardware health evidence.
- Final asset and interface inventory.

## Validation Commands/Tests and Expected Results

| Validation Command/Test | Expected Result |
|---|---|
| iDRAC hardware inventory export and health review | All three servers report healthy status and matching hardware profile |
| BIOS settings review for UEFI, VT-x, VT-d | Virtualization flags and boot mode enabled consistently |
| Firmware inventory comparison against approved baseline | BIOS, iDRAC, NIC, storage controller firmware match the approved version set |
| Remote console and power-cycle test from iDRAC | Each server can be remotely mounted, rebooted, and observed |

## Exit Criteria

- All three servers on the same approved baseline.
- Delivery team can manage each server remotely.

## Rollback / Exit Plan

- If firmware or BIOS changes introduce instability, revert to the previous approved baseline and halt before OS installation.
