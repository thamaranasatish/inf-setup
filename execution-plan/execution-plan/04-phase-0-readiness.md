### FILE: execution-plan/execution-plan/04-phase-0-readiness.md

# Phase 0 — Readiness and Validation

## Objective

Confirm that the fixed architecture, dependencies, installation approach, and capacity constraints are understood, owned, and approved before any server configuration begins.

## Entry Criteria

- Client kickoff completed.
- Dependency owners nominated.
- Fixed architecture and VM placement acknowledged by client stakeholders.

## Steps

1. Review fixed physical server profile, node sizing, and VM placement with platform, network, security, client, and DBA owners.
2. Validate the logical topology of 3 control plane nodes and 3 worker nodes across the 3 physical servers.
3. Validate worker-level capacity math against fixed VM placement.
4. Record the Worker-3 CPU and disk shortfall as a blocking dependency.
5. Record the required client decision for the supported method used to realize six OpenShift nodes on three physical servers.
6. Confirm target OpenShift version, content source model, and VIP delivery method are assigned to owners.
7. Freeze the prerequisite tracker, phase gates, change windows, and sign-off path.

## Outputs

- Signed readiness assessment.
- Dependency tracker with owners and due dates.
- Decision log for unresolved client items.
- Approved implementation calendar.

## Validation Commands/Tests and Expected Results

| Validation Command/Test | Expected Result |
|---|---|
| Prerequisite tracker review against signed checklist | All blocking prerequisites are assigned owners and due phases |
| Capacity workbook review of fixed VM totals versus worker sizing | Worker-1 and Worker-2 fit fixed sizing, Worker-3 shortfall documented with approved disposition |
| Decision log review | Six-node realization method, OpenShift version, and VIP delivery method approved or assigned for closure |
| Kickoff sign-off review | Client, network, security, platform, DBA stakeholders recorded as participants |

## Exit Criteria

- All blocking prerequisites assigned.
- Worker-3 capacity decision closed or formally accepted.
- Six-node realization method approved.

## Rollback / Exit Plan

- If any blocking item remains open, stop execution before infrastructure changes.
- Issue a readiness exception report and reschedule after client decisions are closed.
