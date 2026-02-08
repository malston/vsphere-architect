# Cloud Platform Architecture: VCF and AWS Compared

## The Layered Architecture (Both Platforms)

```
+-------------------------------------+
|         Consumer Services           |  Apps, databases, AI/ML, containers
+-------------------------------------+
|         Platform Services           |  Identity, monitoring, automation, policy
+-------------------------------------+
|      Virtual Infrastructure         |  Compute, storage, networking (software-defined)
+-------------------------------------+
|      Physical Infrastructure        |  Servers, disks, switches, GPUs
+-------------------------------------+
```

Every cloud platform is some variation of this stack. The difference is who owns which layers and how tightly they're integrated.

## How VCF Maps

VCF is a pre-validated, full-stack SDDC -- it bundles the virtual infrastructure layer as a single deployable unit.

| Layer         | VCF Component                       | Role                                                                      |
| ------------- | ----------------------------------- | ------------------------------------------------------------------------- |
| Physical      | Your hardware (VCF-certified)       | You own it, rack it, power it                                             |
| Compute       | vSphere (ESXi + vCenter)            | Hypervisor and VM lifecycle                                               |
| Storage       | vSAN                                | Distributed, software-defined storage across local disks                  |
| Networking    | NSX                                 | Software-defined networking -- microsegmentation, load balancing, routing |
| Orchestration | SDDC Manager                        | Lifecycle management, patching, config drift                              |
| Automation    | Aria Automation (formerly vRealize) | Self-service catalog, IaC, governance                                     |
| Monitoring    | Aria Operations                     | Capacity planning, performance, cost                                      |
| Kubernetes    | Tanzu                               | Container orchestration on top of vSphere                                 |

The key architectural idea: SDDC Manager is the control plane. It treats the entire stack (compute + storage + network) as a single managed domain. You deploy "workload domains" -- isolated resource pools with their own vCenter, NSX, and vSAN clusters -- as the unit of tenancy.

```
SDDC Manager
    |
    +-- Management Domain (runs VCF itself)
    |       +-- vCenter
    |       +-- NSX Manager
    |       +-- vSAN cluster
    |
    +-- Workload Domain A (Production)
    |       +-- vCenter
    |       +-- NSX segments
    |       +-- vSAN cluster
    |
    +-- Workload Domain B (Dev/Test)
            +-- vCenter
            +-- NSX segments
            +-- vSAN cluster
```

## How AWS Maps

AWS is the same layered model, but Amazon owns everything below the API boundary.

| Layer         | AWS Component                  | Role                                           |
| ------------- | ------------------------------ | ---------------------------------------------- |
| Physical      | AWS datacenters                | You never see it                               |
| Compute       | EC2, Lambda, ECS/EKS           | VMs, serverless, containers                    |
| Storage       | EBS, S3, EFS                   | Block, object, file                            |
| Networking    | VPC, Transit Gateway, Route 53 | Software-defined networking, DNS, interconnect |
| Orchestration | CloudFormation, Control Tower  | Stack lifecycle, multi-account governance      |
| Automation    | Service Catalog, CDK           | Self-service, IaC                              |
| Monitoring    | CloudWatch, X-Ray              | Metrics, tracing, logs                         |
| Kubernetes    | EKS                            | Managed Kubernetes                             |

The key architectural idea: the account is the unit of tenancy (analogous to VCF's workload domain). AWS Organizations + Control Tower govern multi-account structures the way SDDC Manager governs workload domains.

```
AWS Organizations
    |
    +-- Management Account (billing, governance)
    |
    +-- Production OU
    |       +-- Account: prod-app-a (VPC, EC2, RDS)
    |       +-- Account: prod-app-b (VPC, EKS, S3)
    |
    +-- Dev/Test OU
            +-- Account: dev-app-a
            +-- Account: dev-app-b
```

## The Conceptual Parallels

| Concern             | VCF                        | AWS                                      |
| ------------------- | -------------------------- | ---------------------------------------- |
| Tenancy boundary    | Workload Domain            | AWS Account                              |
| Control plane       | SDDC Manager               | Control Tower / Organizations            |
| Network isolation   | NSX segments + firewalls   | VPCs + security groups                   |
| Storage abstraction | vSAN policies              | EBS/S3 storage classes                   |
| IaC                 | Aria Automation, Terraform | CloudFormation, CDK, Terraform           |
| Container platform  | Tanzu (TKG)                | EKS                                      |
| Identity            | vCenter SSO + AD           | IAM + SSO (Identity Center)              |
| Lifecycle/patching  | SDDC Manager bundles       | AWS manages it (your problem for EC2 OS) |

## The Fundamental Difference

VCF gives you the entire stack to operate. You get full control and full responsibility. You decide the hardware, the topology, the upgrade cadence. The trade-off is operational burden.

AWS gives you the API surface. Everything below is someone else's problem. The trade-off is less control, vendor lock-in, and costs that scale with consumption rather than capacity.

Both architectures converge on the same principles: software-defined everything, policy-driven automation, declarative infrastructure, and isolation through logical boundaries rather than physical ones.
