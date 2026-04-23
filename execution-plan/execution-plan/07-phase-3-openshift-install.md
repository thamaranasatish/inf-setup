### FILE: execution-plan/execution-plan/07-phase-3-openshift-install.md

# Phase 3 — OpenShift Installation (3 Control Plane + 3 Worker Nodes)

## Objective

Install a six-node OpenShift cluster consisting of three dedicated control plane nodes and three dedicated worker nodes. Control plane nodes must remain dedicated and not schedulable for user workloads or VMs.

## Entry Criteria

- Phase 2 completed.
- Client-approved OpenShift version selected.
- Pull secret and content source available.
- Six-node realization method approved.

## Steps

1. Prepare the bastion host with approved OpenShift installation binaries and access credentials.
2. Create the installation configuration using the approved cluster name, base domain, and networking inputs.
3. Define the six node identities: three dedicated control plane nodes and three dedicated worker nodes.
4. Execute the client-approved Red Hat-supported installation workflow for the six-node layout.
5. Confirm the cluster scheduler is configured with `mastersSchedulable=false` so control plane nodes remain dedicated and not schedulable for user workloads.
6. Confirm control plane nodes retain the expected `NoSchedule` taints and that no user workloads or VMs can land on them.
7. Confirm all cluster operators reach stable state.
8. Establish baseline administrative access from the bastion host.

## Outputs

- Running OpenShift cluster.
- Bastion administration access.
- Initial cluster configuration backup.
- Evidence that control plane nodes are dedicated and not schedulable.

## Validation Commands/Tests and Expected Results

| Validation Command/Test | Expected Result |
|---|---|
| `oc get nodes` | Exactly 3 control plane nodes and 3 worker nodes show `Ready` state |
| `oc get schedulers.config.openshift.io cluster -o jsonpath='{.spec.mastersSchedulable}'` | Returns `false` |
| `oc get nodes -l node-role.kubernetes.io/master -o custom-columns=NAME:.metadata.name,UNSCHEDULABLE:.spec.unschedulable` | All control plane nodes show dedicated unschedulable state |
| `oc describe node <control-plane-node> | grep -i Taints` | Each control plane node includes a `NoSchedule` taint and is not available for VM or user workload placement |
| `oc get co` | Core cluster operators report `Available=True` and no blocking degraded state |
| Bastion-to-API login test using `oc login` | Cluster API reachable from the approved admin network |

## Exit Criteria

- Six-node cluster is stable.
- Admin access works from bastion.
- Control plane nodes confirmed dedicated and not schedulable for user workloads or VMs.

## Rollback / Exit Plan

- If installation fails or cluster health is unstable, destroy the partial deployment using the approved method, preserve logs, and restart only after blocker resolution.
