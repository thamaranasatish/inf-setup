### FILE: execution-plan/11-phase-7b-vm-placement-affinity-anti-affinity.md

# Phase 7b — VM Placement, Affinity, and Anti-Affinity

## Objective
Pin each VM to its designated worker per the fixed placement and avoid scheduling to control plane nodes.

## Steps

1. Label worker nodes with intent-based labels (e.g., `workload-tier=cicd`, `workload-tier=data`, `workload-tier=observability`) for use by VM node selectors.
   - Official doc: https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/nodes/working-with-nodes

2. For each VM, set `spec.template.spec.nodeSelector` (or `affinity` / `nodeAffinity`) targeting the correct worker per the placement map in `01-architecture-baseline.md`.
   - Official doc: https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/virtualization/virtual-machines#virt-specifying-node-affinity-rules-for-vms
   - Official doc: https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/nodes/controlling-pod-placement-onto-nodes-scheduling

3. Do not add tolerations that allow scheduling to control plane nodes. Control plane `NoSchedule` taints must remain unbypassed for VM workloads.
   - Official doc: https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/nodes/controlling-pod-placement-onto-nodes-scheduling

4. For ELK instances (VM8 and VM9) on Worker-3, apply anti-affinity within the same worker per Elastic’s guidance where applicable at the application layer.
   - Official doc: https://www.elastic.co/guide/en/elasticsearch/reference/current/high-availability-cluster-small-clusters.html

## Validation / Expected Results

- `oc get vmi -A -o custom-columns=VM:.metadata.name,NODE:.status.nodeName` matches the fixed placement map.
- `oc describe vm <vm-name>` shows the expected affinity / nodeSelector, no tolerations bypassing control plane taints.
  - Official doc: https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/virtualization/virtual-machines

## Exit Criteria

- Every VM placed exactly on its assigned worker; control plane remains free of VMs.
