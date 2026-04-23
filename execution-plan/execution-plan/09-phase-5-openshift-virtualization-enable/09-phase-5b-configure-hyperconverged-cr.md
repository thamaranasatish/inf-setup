### FILE: execution-plan/09-phase-5b-configure-hyperconverged-cr.md

# Phase 5b — Configure the HyperConverged CR

## Objective
Deploy and validate the HyperConverged custom resource (HCO) that activates KubeVirt, CDI, and supporting components.

## Steps

1. Create the HyperConverged CR in `openshift-cnv` using the default configuration provided by the Operator, unless an approved deviation is required.
   - Official doc: https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/virtualization/installing

2. Confirm the HCO rolls out KubeVirt, CDI (Containerized Data Importer), SSP, and networking components on worker nodes.
   - Official doc: https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/virtualization/installing

3. Confirm no virtualization workloads target control plane nodes. Control plane taints remain in place and must not be tolerated by virtualization workloads.
   - Official doc: https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/nodes/controlling-pod-placement-onto-nodes-scheduling

4. Install `virtctl` on the bastion per the official CLI guide.
   - Official doc: https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/virtualization/virtctl

## Validation / Expected Results

- `oc get hyperconverged -n openshift-cnv` shows `Available=True`.
- `oc get pods -A -o wide | grep -E 'virt-|kubevirt'` shows virtualization pods only on worker nodes.
- `virtctl version` returns client and server versions.
  - Official doc: https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/virtualization/virtctl

## Exit Criteria

- HyperConverged healthy; virtualization stack scheduled only to workers.
