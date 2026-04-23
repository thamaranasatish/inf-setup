### FILE: execution-plan/09-phase-5a-install-openshift-virtualization-operator.md

# Phase 5a — Install the OpenShift Virtualization Operator

## Objective
Install the Red Hat OpenShift Virtualization Operator from the approved catalog source.

## Steps

1. Verify cluster meets OpenShift Virtualization prerequisites (hardware CPU virtualization, supported OCP version, sufficient resources on worker nodes).
   - Official doc: https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/virtualization/installing

2. Install the Operator from the web console via **Operators → OperatorHub → OpenShift Virtualization** into the `openshift-cnv` namespace using the approved update channel (e.g., `stable`).
   - Official doc: https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/virtualization/installing

3. Or install via CLI by creating the `Namespace`, `OperatorGroup`, and `Subscription` objects as documented.
   - Official doc: https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/virtualization/installing

## Validation / Expected Results

- `oc get csv -n openshift-cnv` shows `kubevirt-hyperconverged-operator.*` in `Succeeded`.
- `oc get pods -n openshift-cnv` shows operator pods `Running`.
  - Official doc: https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/virtualization/installing

## Exit Criteria

- Operator installed and subscribed on the approved channel.
