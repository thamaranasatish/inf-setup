### FILE: execution-plan/07-phase-3b-post-install-smoke-tests.md

# Phase 3b — Post-Install Smoke Tests

## Objective
Verify the newly installed cluster is healthy and matches the required topology.

## Steps

1. Inspect nodes and cluster operators:
   ```
   oc get nodes
   oc get clusterversion
   oc get co
   ```
   - Official doc: https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/cli_tools/openshift-cli-oc

2. Verify control plane dedication:
   ```
   oc get schedulers.config.openshift.io cluster -o jsonpath='{.spec.mastersSchedulable}'
   oc get nodes -l node-role.kubernetes.io/master -o custom-columns=NAME:.metadata.name,UNSCHEDULABLE:.spec.unschedulable
   ```
   - Official doc: https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/nodes/working-with-clusters

3. Validate ingress and API VIPs respond over HTTPS:
   ```
   curl -skI https://api.<cluster>.<baseDomain>:6443
   curl -skI https://console-openshift-console.apps.<cluster>.<baseDomain>
   ```
   - Official doc: https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/web_console/index

4. Confirm monitoring stack is up:
   ```
   oc get co monitoring
   oc get pods -n openshift-monitoring
   ```
   - Official doc: https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/monitoring/index

## Validation / Expected Results

- Nodes Ready, operators Available, VIPs respond, monitoring healthy.

## Exit Criteria

- Cluster accepted as ready for Phase 4 baseline.
