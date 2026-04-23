### FILE: execution-plan/execution-plan/09-phase-5-openshift-virtualization-enable.md

# Phase 5 — OpenShift Virtualization Enablement

## Objective

Enable Red Hat OpenShift Virtualization on worker nodes so that all application workloads can run as VMs alongside containerized workloads on the same OpenShift platform. Control plane nodes must remain dedicated and not schedulable.

## Entry Criteria

- Phase 4 completed.
- Cluster operators healthy.
- Worker nodes available for virtualization workloads.

## Steps

1. Install the Red Hat OpenShift Virtualization Operator from the approved catalog source.
2. Create and validate the HyperConverged custom resource.
3. Confirm KubeVirt and supporting components are running successfully on worker nodes only.
4. Define virtualization namespaces, role bindings, and administration boundaries.
5. Confirm VM workloads cannot schedule to control plane nodes.
6. Validate `virtctl` access and test a minimal VM lifecycle operation.

## Outputs

- OpenShift Virtualization-enabled cluster.
- HyperConverged configuration.
- Virtualization administration model.

## Validation Commands/Tests and Expected Results

| Validation Command/Test | Expected Result |
|---|---|
| `oc get csv -A | grep -i kubevirt` | OpenShift Virtualization Operator installation succeeded |
| `oc get hyperconverged -A` | HyperConverged custom resource reports healthy status |
| `oc get pods -A -o wide | grep -E 'virt-|kubevirt'` | Virtualization pods run only on worker nodes |
| Attempt to schedule a test VM onto a control plane node via nodeSelector | Scheduling is blocked by control plane taints |
| `virtctl start <smoke-vm>` followed by `oc get vmi -A -o wide` | Smoke VM enters `Running` state on a worker and stops cleanly when requested |

## Exit Criteria

- OpenShift Virtualization healthy and ready for storage configuration.
- Control plane nodes confirmed free of any VM workload.

## Rollback / Exit Plan

- If virtualization enablement fails before VM onboarding, remove the operator and related resources, collect logs, and return the cluster to the Phase 4 baseline.
