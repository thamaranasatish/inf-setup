### FILE: execution-plan/05-phase-1a-bios-firmware-settings.md

# Phase 1a — BIOS and Firmware Settings (Dell PowerEdge R750)

## Objective
Bring each R750 to a supportable BIOS and firmware baseline with virtualization features enabled.

## Steps

1. Update BIOS, iDRAC, NIC, PERC/HBA, and SSD firmware to the Dell-approved baseline using Dell Lifecycle Controller / iDRAC firmware update.
   - Official doc: https://www.dell.com/support/manuals/en-us/idrac9-lifecycle-controller-v5.x-series/idrac9_5.xx_ug/updating-firmware-using-idrac-with-lifecycle-controller

2. Set boot mode to UEFI for all three servers.
   - Official doc: https://www.dell.com/support/manuals/en-us/poweredge-r750/r750_bios_ref

3. Enable Intel VT-x (Virtualization Technology) and VT-d (I/O virtualization) in BIOS Processor / Integrated Devices settings.
   - Official doc: https://www.dell.com/support/manuals/en-us/poweredge-r750/r750_bios_ref

4. Configure consistent Hyper-Threading / Logical Processor setting across all three servers per client baseline.
   - Official doc: https://www.dell.com/support/manuals/en-us/poweredge-r750/r750_bios_ref

5. Apply the client-approved local-SSD presentation model (RAID or HBA mode) via PERC controller.
   - Official doc: https://www.dell.com/support/manuals/en-us/perc-h755-series/perc_h755_ug

## Validation / Expected Results

- `racadm get BIOS.ProcSettings.ProcVirtualization` returns `Enabled` on each server.
  - Official doc: https://www.dell.com/support/manuals/en-us/idrac9-lifecycle-controller-v5.x-series/idrac9_5.xx_racadm
- BIOS settings export from each server matches the approved baseline.
- Firmware inventory from iDRAC matches approved versions.

## Exit Criteria

- All three servers on matching BIOS and firmware baseline.
- Virtualization extensions enabled and evidenced.
