### FILE: execution-plan/07-phase-3a-openshift-install-baremetal-shell.md

# Phase 3a — OpenShift Bare-Metal Install (Step-by-Step)

## Objective
Install a 3 control plane + 3 worker OpenShift cluster on three Dell EMC PowerEdge R750 servers, using the Red Hat-supported Agent-Based Installer for bare metal. Control plane nodes remain dedicated and not schedulable.

> Choice of installer: this plan uses the Agent-Based Installer per Red Hat’s recommended path for disconnected / bare-metal footprints. If the client has formally chosen Assisted Installer or Installer-Provisioned Infrastructure (IPI) for bare metal, the equivalent official procedure below applies and will be followed exactly.
> - Agent-Based: https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/installing_an_on-premise_cluster_with_the_agent-based_installer/index
> - Assisted Installer: https://docs.redhat.com/en/documentation/assisted_installer_for_openshift_container_platform
> - IPI on bare metal: https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/installing_on_bare_metal/index

## Prerequisites

- Phases 1 and 2 complete.
- RHEL bastion host prepared with SSH key and CLI tools.
- Red Hat pull secret available.

## Steps

1. On the RHEL bastion, install the OpenShift client (`oc`) and the agent installer binary (`openshift-install`) for the approved OCP version.
   - Official doc: https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/cli_tools/openshift-cli-oc
   - Official doc: https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/installing_an_on-premise_cluster_with_the_agent-based_installer/preparing-to-install-with-agent-based-installer

2. Generate an SSH key pair on the bastion to be embedded in RHCOS installs (`ssh-keygen -t ed25519`).
   - Official doc: https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/installing_an_on-premise_cluster_with_the_agent-based_installer/preparing-to-install-with-agent-based-installer

3. Create `install-config.yaml` for 3 control plane + 3 worker bare-metal nodes, specifying cluster name, base domain, API VIP, Ingress VIP, machine CIDR, service CIDR, cluster network CIDR, and pull secret.
   - Official doc: https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/installing_an_on-premise_cluster_with_the_agent-based_installer/preparing-to-install-with-agent-based-installer

4. Create `agent-config.yaml` listing the three control plane nodes and three worker nodes with their MAC addresses, hostnames, and network configuration (static or DHCP per approved design).
   - Official doc: https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/installing_an_on-premise_cluster_with_the_agent-based_installer/preparing-to-install-with-agent-based-installer

5. Generate the agent ISO:
   ```
   openshift-install agent create image --dir <assets-dir>
   ```
   - Official doc: https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/installing_an_on-premise_cluster_with_the_agent-based_installer/installing-with-agent-based-installer

6. For each server, mount `agent.x86_64.iso` as virtual media via iDRAC, set one-time boot to the virtual CD, and power on.
   - Official doc: https://www.dell.com/support/manuals/en-us/idrac9-lifecycle-controller-v5.x-series/idrac9_5.xx_ug
   - Official doc: https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/installing_an_on-premise_cluster_with_the_agent-based_installer/installing-with-agent-based-installer

7. Monitor bootstrap and install progress from the bastion:
   ```
   openshift-install --dir <assets-dir> agent wait-for bootstrap-complete --log-level=info
   openshift-install --dir <assets-dir> agent wait-for install-complete --log-level=info
   ```
   - Official doc: https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/installing_an_on-premise_cluster_with_the_agent-based_installer/installing-with-agent-based-installer

8. Configure `kubeconfig` and verify cluster operator status:
   ```
   export KUBECONFIG=<assets-dir>/auth/kubeconfig
   oc get nodes
   oc get clusterversion
   oc get co
   ```
   - Official doc: https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/installing_an_on-premise_cluster_with_the_agent-based_installer/installing-with-agent-based-installer

9. Enforce control plane dedication. On OCP the default scheduler policy keeps control plane nodes non-schedulable; confirm and do not change:
   ```
   oc get schedulers.config.openshift.io cluster -o jsonpath='{.spec.mastersSchedulable}'
   ```
   Expected `false`.
   - Official doc: https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/nodes/working-with-clusters#nodes-nodes-working-master-schedulable_nodes-nodes-working

10. Approve any pending worker CSRs (if present) using `oc adm certificate approve` per the install guide.
    - Official doc: https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/installing_an_on-premise_cluster_with_the_agent-based_installer/installing-with-agent-based-installer

## Validation / Expected Results

- `oc get nodes` reports exactly 3 control plane nodes and 3 worker nodes in `Ready` state.
- `oc get co` reports all cluster operators `Available=True` and not `Degraded`.
- `oc get schedulers.config.openshift.io cluster -o jsonpath='{.spec.mastersSchedulable}'` returns `false`.
  - Official doc: https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/nodes/working-with-clusters
- `oc describe node <control-plane-node> | grep -i Taints` shows a `NoSchedule` taint on each control plane node.
  - Official doc: https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/nodes/controlling-pod-placement-onto-nodes-scheduling

## Exit Criteria

- Cluster fully installed and healthy.
- Control plane confirmed dedicated and not schedulable.
- `kubeconfig`, pull secret, and installation artifacts archived securely.
