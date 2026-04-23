### FILE: execution-plan/04-phase-0a-readiness-prechecks.md

# Phase 0a — Readiness Pre-Checks

## Objective
Confirm fixed architecture, dependencies, and capacity constraints are owned and approved before any build activity.

## Steps

1. Confirm each Dell EMC PowerEdge R750 server matches the fixed hardware profile (2×Xeon Silver 4310, 2 TB RAM, 2 TB SSD local).
   - Official doc: https://www.dell.com/support/home/en-us/product-support/product/poweredge-r750/docs

2. Confirm the supported Red Hat OpenShift Container Platform version and the matching OpenShift Virtualization version are selected.
   - Official doc: https://access.redhat.com/support/policy/updates/openshift
   - Official doc: https://docs.redhat.com/en/documentation/openshift_container_platform

3. Confirm the fixed six-node topology (3 dedicated control plane + 3 worker nodes) is approved as documented in the cluster installation reference.
   - Official doc: https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/installing/index

4. Record the Worker-3 placement capacity exception (28 vCPU, 1,780 GB disk vs fixed 20 vCPU / ~1.3 TB usable) in the decision log. Client must disposition before Phase 6/7.
   - OFFICIAL DOC NOT FOUND – NEEDS CLIENT CONFIRMATION for the chosen overcommit / capacity exception policy.

## Validation / Expected Results

- Signed readiness assessment present.
- OCP and OpenShift Virtualization versions explicitly approved in writing.
- Worker-3 capacity exception recorded with an owner and due date.

## Exit Criteria

- All blocking prerequisites assigned.
- Six-node realization method approved.
- Worker-3 capacity disposition approved or formally accepted.
