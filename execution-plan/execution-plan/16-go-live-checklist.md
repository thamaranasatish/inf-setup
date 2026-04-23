### FILE: execution-plan/execution-plan/16-go-live-checklist.md

# 16 Go-Live and Handover Readiness Checklist

## 16.1 Platform Health

- [ ] Three control plane nodes `Ready` and not schedulable for user workloads.
- [ ] `oc get schedulers.config.openshift.io cluster -o jsonpath='{.spec.mastersSchedulable}'` returns `false`.
- [ ] Each control plane node shows `UNSCHEDULABLE=true` and the expected `NoSchedule` taint.
- [ ] Three worker nodes `Ready` and host only the intended VM workloads.
- [ ] OpenShift Virtualization Operator and HyperConverged healthy.
- [ ] API VIP and Ingress VIP reachable from approved client networks.

## 16.2 Security Sign-Offs

- [ ] Identity integration validated with approved test accounts.
- [ ] RBAC matrix signed off and enforced.
- [ ] TLS certificates deployed and within validity period.
- [ ] Audit logging and access logging forwarded to approved SIEM/syslog.
- [ ] Secrets integration from VM3 CI/CD to VM12 validated (no cleartext credentials on VM3).
- [ ] PAM, MFA, and break-glass processes operational.
- [ ] Vulnerability scanning and EDR coverage applied per approved scope.

## 16.3 Operational Readiness

- [ ] Named support owners and escalation paths agreed.
- [ ] Bastion access, privileged access, and change control process handed over.
- [ ] Standard operating procedures for VM restart, cluster access, and local-storage limitations documented.
- [ ] Known DEV limitations reviewed and accepted with client stakeholders, including local-storage mobility constraints.

## 16.4 Monitoring and Logging Readiness

- [ ] Cluster monitoring baseline healthy.
- [ ] ELK and Grafana ingest required logs and metrics.
- [ ] Critical alerts and dashboard ownership assigned.

## 16.5 Backup and Restore Readiness (DEV Scope)

- [ ] Client-approved DEV backup, export, or rebuild strategy documented.
- [ ] Recovery expectations for node-local VM disks reviewed and accepted.
- [ ] Configuration exports and credential custody process completed.

## 16.6 Handover Artifacts List

- [ ] Final execution plan (this folder).
- [ ] Logical architecture and network topology diagrams.
- [ ] Build inventory of servers, nodes, VMs, IPs, DNS records, and VIPs.
- [ ] RBAC matrix and access ownership record.
- [ ] Operational runbooks and support contacts.
- [ ] Backup or export policy for DEV.
- [ ] Defect list, risk register, and accepted exceptions (including Worker-3 capacity disposition).
