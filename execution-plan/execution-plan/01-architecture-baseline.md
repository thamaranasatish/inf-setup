### FILE: execution-plan/execution-plan/01-architecture-baseline.md

# 01 Architecture Baseline

## Physical Infrastructure (Fixed)

- Three identical Dell EMC PowerEdge R750 servers.
- CPU per server: 2 × Intel Xeon Silver 4310, 24 physical cores (48 threads).
- RAM per server: 64 × 32 GB RDIMM = 2 TB.
- Local storage per server: 2 TB SSD. No shared storage in DEV scope.

## Cluster Topology (Fixed)

- 3 dedicated Control Plane nodes.
- 3 dedicated Worker nodes.
- Each physical server hosts one control plane node and one worker node.
- Control plane nodes are not schedulable for user workloads or VMs.

## Node Sizing (Fixed)

| Role | vCPU | RAM | Disk |
|---|---|---|---|
| Control Plane (each) | 8 | 64 GB | 300 GB |
| Worker (each) | 20 | 512 GB | ~1.3 TB usable |

## Application VM Placement (Fixed)

| Worker | VM | Purpose | vCPU | RAM | Disk |
|---|---|---|---|---|---|
| Worker-1 | VM3 | CI/CD Stack: Jenkins, Nexus, SonarQube, Aqua, Checkmarx, Redgate, Maven, JDK, Newman, NodeJS | 16 | 96 GB | 600 GB |
| Worker-1 | VM12 | Secrets: HashiCorp Vault OR CyberArk | 4 | 16 GB | 150 GB |
| Worker-2 | VM10 | MS SQL Server 2022 | 16 | 128 GB | 800 GB |
| Worker-2 | VM11 | ActiveMQ (existing) | 4 | 16 GB | 200 GB |
| Worker-3 | VM6 | F5 NGINX API Gateway | 4 | 16 GB | 80 GB |
| Worker-3 | VM7 | Kafka + Cache | 8 | 64 GB | 500 GB |
| Worker-3 | VM8 | ELK + Grafana (Instance 1) | 8 | 64 GB | 600 GB |
| Worker-3 | VM9 | ELK + Grafana (Instance 2) | 8 | 64 GB | 600 GB |

### Worker Aggregate Observation

| Worker | Aggregate vCPU | Aggregate RAM | Aggregate Disk | Fixed Worker Capacity Observation |
|---|---|---|---|---|
| Worker-1 | 20 | 112 GB | 750 GB | Fits fixed worker CPU and disk |
| Worker-2 | 20 | 144 GB | 1,000 GB | Fits fixed worker CPU and disk |
| Worker-3 | 28 | 208 GB | 1,780 GB | Exceeds fixed worker CPU by 8 vCPU and fixed usable disk by approximately 480 GB — requires client disposition |

## Logical Architecture

```mermaid
flowchart TB
    subgraph DC[On-Prem DEV OpenShift Platform]
        subgraph S1[Physical Server 1]
            CP1[Control Plane 1<br/>Dedicated, Not Schedulable]
            W1[Worker 1]
            W1 --> VM3[VM3 CI/CD]
            W1 --> VM12[VM12 Secrets]
        end
        subgraph S2[Physical Server 2]
            CP2[Control Plane 2<br/>Dedicated, Not Schedulable]
            W2[Worker 2]
            W2 --> VM10[VM10 SQL Server 2022]
            W2 --> VM11[VM11 ActiveMQ]
        end
        subgraph S3[Physical Server 3]
            CP3[Control Plane 3<br/>Dedicated, Not Schedulable]
            W3[Worker 3]
            W3 --> VM6[VM6 NGINX API Gateway]
            W3 --> VM7[VM7 Kafka and Cache]
            W3 --> VM8[VM8 ELK and Grafana 1]
            W3 --> VM9[VM9 ELK and Grafana 2]
        end
        CP1 --- CP2
        CP2 --- CP3
    end

    DNS[DNS]
    NTP[NTP]
    IDP[AD / LDAP / SSO]
    APIVIP[API VIP]
    INGVIP[Ingress VIP]
    DNS --> APIVIP
    DNS --> INGVIP
    NTP --> DC
    IDP --> DC
```

## Network Topology

```mermaid
flowchart LR
    Users[Developers, QA, Security Users]
    Admin[Platform and Delivery Admins]
    OOB[OOB Management<br/>iDRAC]
    Jump[RHEL Bastion]
    FW[Firewall Controls]
    VIP[API VIP and Ingress VIP]
    DNS[DNS]
    NTP[NTP]
    IDP[AD / LDAP / SSO]

    subgraph OCP[OpenShift Cluster]
        CP[3 Dedicated Control Plane Nodes]
        WK[3 Worker Nodes]
        VMS[OpenShift Virtualization VMs]
        CP --> WK
        WK --> VMS
    end

    Admin --> OOB
    Admin --> Jump
    Users --> FW
    Jump --> FW
    FW --> VIP
    VIP --> OCP
    DNS --> VIP
    DNS --> OCP
    NTP --> OCP
    IDP --> OCP
```

## Hypervisor Confirmation

- OpenShift Virtualization is the default and preferred virtualization layer for all application VMs.
- No external hypervisor required for application VM execution.
- No VMware, ESXi, or Hyper-V is assumed anywhere in this plan.
