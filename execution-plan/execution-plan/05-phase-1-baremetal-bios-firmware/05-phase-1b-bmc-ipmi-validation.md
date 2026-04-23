### FILE: execution-plan/05-phase-1b-bmc-ipmi-validation.md

# Phase 1b — BMC / iDRAC Validation

## Objective
Validate out-of-band management is operational for install and Day-2 operations.

## Steps

1. Assign dedicated iDRAC IP addresses on the OOB management network for each server.
   - Official doc: https://www.dell.com/support/manuals/en-us/idrac9-lifecycle-controller-v5.x-series/idrac9_5.xx_ug

2. Configure iDRAC users, roles, and service accounts per client least-privilege policy.
   - Official doc: https://www.dell.com/support/manuals/en-us/idrac9-lifecycle-controller-v5.x-series/idrac9_5.xx_ug

3. Validate virtual console and virtual media access from the jump host to each iDRAC.
   - Official doc: https://www.dell.com/support/manuals/en-us/idrac9-lifecycle-controller-v5.x-series/idrac9_5.xx_ug

4. Test remote power-cycle via iDRAC on each server.
   - Official doc: https://www.dell.com/support/manuals/en-us/idrac9-lifecycle-controller-v5.x-series/idrac9_5.xx_racadm

## Validation / Expected Results

- `racadm serveraction powerstatus` returns `ON` for each server.
- Virtual console session can be launched and a virtual media ISO can be mounted.

## Exit Criteria

- BMC access confirmed for all three servers and recorded in the build inventory.
