### FILE: execution-plan/execution-plan/14-runbooks-validation-tests.md

# 14 Runbooks — Consolidated Validation Commands

This file consolidates the phase-level validation commands for quick execution during the delivery runs and handover. All commands are executed from the bastion unless otherwise stated.

## 14.1 Cluster Readiness

```bash
oc get nodes
oc get nodes -o wide
oc get co
oc get clusterversion
oc whoami --show-server
```

Expected: 3 `Ready` control plane nodes and 3 `Ready` worker nodes; all cluster operators `Available=True`.

## 14.2 Dedicated Control Plane — NOT Schedulable

```bash
oc get schedulers.config.openshift.io cluster -o jsonpath='{.spec.mastersSchedulable}'
oc get nodes -l node-role.kubernetes.io/master \
  -o custom-columns=NAME:.metadata.name,UNSCHEDULABLE:.spec.unschedulable,TAINTS:.spec.taints
for n in $(oc get nodes -l node-role.kubernetes.io/master -o name); do
  echo "==> $n"; oc describe "$n" | grep -i Taints -A2
done
```

Expected: `mastersSchedulable` returns `false`; every control plane node reports `UNSCHEDULABLE=true` and a `NoSchedule` taint; no user workload or VM is placed on control plane nodes.

## 14.3 Network, DNS, NTP, Egress

```bash
nslookup api.<cluster>.<baseDomain>
nslookup api-int.<cluster>.<baseDomain>
nslookup test.apps.<cluster>.<baseDomain>
chronyc sources -v
nc -vz <mirror-or-registry-endpoint> 443
nc -vz <idp-endpoint> 443
```

Expected: API and API-int resolve to API VIP; wildcard resolves to Ingress VIP; NTP selected; outbound HTTPS paths open.

## 14.4 Identity, RBAC, TLS, Audit

```bash
oc login -u <approved-identity-user>
oc auth can-i '*' '*' --as <test-admin-group>
oc adm policy who-can get pods -n openshift-cnv
openssl s_client -connect api.<cluster>.<baseDomain>:6443 -showcerts </dev/null
oc get clusterlogforwarder -A
```

Expected: SSO login succeeds, RBAC shows only approved groups with elevated rights, TLS chain matches approved CA, audit logs forwarded.

## 14.5 OpenShift Virtualization

```bash
oc get csv -A | grep -i kubevirt
oc get hyperconverged -A
oc get pods -A -o wide | grep -E 'virt-|kubevirt'
virtctl version
```

Expected: Operator healthy, HyperConverged Available, virtualization pods only on workers.

## 14.6 Local Storage

```bash
oc get storageclass
oc get pv,pvc -A
oc describe pvc <test-pvc>
```

Expected: Approved worker-local storage class present; test PVC binds to intended worker.

## 14.7 VM Placement and Health

```bash
oc get vm,vmi -A -o wide
oc get vmi -A -o custom-columns=VM:.metadata.name,NODE:.status.nodeName,STATUS:.status.phase
for vm in VM3 VM12 VM10 VM11 VM6 VM7 VM8 VM9; do
  oc describe vm "$vm" -n <vm-namespace> | grep -A5 'Node Selector\|Affinity'
done
```

Expected: Each VM runs only on its designated worker, matches placement map.

## 14.8 Application Smoke Tests

```bash
# SQL Server 2022 (VM10)
sqlcmd -S <sql-host> -Q "SELECT @@VERSION"

# API Gateway (VM6)
curl -sk https://<nginx-host>/health

# Observability (VM8 / VM9)
curl -s http://<elasticsearch-host>:9200/_cluster/health
curl -sk https://<grafana-host>/api/health
```

Expected: Each application endpoint returns the documented healthy response.

## 14.9 Handover Smoke Run

- Trigger a CI/CD pipeline on VM3 that pulls a secret from VM12, builds an artifact, publishes to Nexus, scans in SonarQube/Aqua/Checkmarx, and emits logs and metrics visible in ELK and Grafana on VM8/VM9.
- Exercise an ActiveMQ and Kafka produce/consume round-trip.
- Run a SQL Server 2022 connectivity check from VM3.
- Confirm all events reached the SIEM or syslog destination.
