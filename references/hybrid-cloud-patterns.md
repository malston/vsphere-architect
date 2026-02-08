# Hybrid Cloud Patterns

## What Hybrid Cloud Means in Practice

Hybrid cloud is workloads spanning on-prem and public cloud infrastructure with some degree of shared management, networking, or data plane. It's not just "we have on-prem and also use AWS" -- that's multi-cloud. Hybrid implies connectivity and workload portability between the two.

## VMware-Native Hybrid Options

### VMware Cloud on AWS (VMC on AWS)

A VMware SDDC running on bare-metal AWS infrastructure, operated by Broadcom/VMware.

```
On-Prem vSphere                     VMC on AWS
+-------------------+               +-------------------+
| vCenter           |               | vCenter (cloud)   |
| ESXi hosts        |  <-- HCX -->  | ESXi on AWS metal |
| vSAN              |               | vSAN              |
| NSX               |               | NSX               |
+-------------------+               +-------------------+
        |                                    |
    On-prem network  <-- VPN/DX -->  AWS VPC
```

**What you get:**

- Same vSphere APIs, same tools, same operational model
- vMotion between on-prem and cloud (via HCX)
- NSX networking consistent across both
- Access to native AWS services from cloud-side VMs

**Architecture details:**

- Minimum 2 hosts (i3.metal or i4i.metal instances)
- Dedicated bare-metal -- not shared. You get real ESXi on real hardware.
- vCenter in the cloud SDDC is managed by VMware (patching, upgrades)
- Storage: vSAN on local NVMe + optional elastic vSAN using AWS EBS
- Networking: NSX overlay connected to AWS VPC via ENI

**When to use:**

- DR site without building a second datacenter
- Cloud burst for temporary capacity
- Data center evacuation or migration staging
- Accessing AWS services (ML, analytics) from VMware workloads

**When it doesn't fit:**

- Cost-sensitive steady-state workloads (VMC pricing is premium)
- Small footprint (minimum 2 hosts is a significant commitment)
- Workloads that should be cloud-native (containers, serverless)

### Azure VMware Solution (AVS)

Same concept as VMC on AWS but running on Azure bare-metal infrastructure.

```
On-Prem vSphere                     Azure VMware Solution
+-------------------+               +-------------------+
| vCenter           |               | vCenter (AVS)     |
| ESXi hosts        |  <-- HCX -->  | ESXi on Azure     |
| vSAN              |               | vSAN              |
| NSX               |               | NSX-T             |
+-------------------+               +-------------------+
        |                                    |
    On-prem network  <-- VPN/ER -->  Azure VNet
```

**Key differences from VMC on AWS:**

- Runs on Azure dedicated hosts
- ExpressRoute for connectivity (vs Direct Connect on AWS)
- Tight integration with Azure native services (AAD, Monitor, Security Center)
- Billed and supported through Microsoft (not Broadcom directly)

**When to choose AVS over VMC on AWS:**

- Already an Azure shop (existing ExpressRoute, Azure AD, Azure subscriptions)
- Microsoft EA agreements with committed spend
- Need Azure-native service integration more than AWS services

### Google Cloud VMware Engine (GCVE)

VMware SDDC on Google Cloud bare-metal. Same pattern as VMC and AVS.

**When to choose GCVE:**

- Google Cloud is your primary cloud
- BigQuery, Vertex AI, or other Google services are key workload dependencies
- Existing Google Cloud interconnect

### Choosing Between VMware Cloud Providers

| Factor                     | VMC on AWS             | AVS               | GCVE               |
| -------------------------- | ---------------------- | ----------------- | ------------------ |
| Primary cloud relationship | AWS                    | Azure             | Google             |
| Connectivity               | Direct Connect         | ExpressRoute      | Cloud Interconnect |
| Billing                    | Broadcom               | Microsoft         | Google             |
| Native service integration | AWS services           | Azure services    | Google services    |
| Minimum hosts              | 2                      | 3                 | 3                  |
| Operational model          | VMware-managed vCenter | Microsoft-managed | Google-managed     |

The answer is almost always "whichever cloud you already use." The VMware layer is the same -- vSphere, vSAN, NSX -- the differentiator is which cloud services you want adjacent.

## VMware HCX (Hybrid Cloud Extension)

HCX is the migration and connectivity layer for hybrid VMware environments. It's how you move workloads between on-prem and cloud.

### What HCX Does

| Capability          | How It Works                                                    |
| ------------------- | --------------------------------------------------------------- |
| Bulk migration      | Parallel VM migration using vSphere Replication                 |
| vMotion migration   | Live migration with zero downtime (network extension required)  |
| Network extension   | Stretch L2 networks between sites (VMs keep their IP addresses) |
| WAN optimization    | Deduplication and compression over the WAN link                 |
| Traffic engineering | Route control for migrated workloads                            |

### Migration Types

```
                    HCX Migration Options
                           |
        +------------------+------------------+
        |                  |                  |
   Bulk Migration    vMotion Migration   Replication
                                         Assisted vMotion
   - Background         - Zero downtime     - Combines both
   - VM reboots at      - Network extension - Sync via replication
     destination           required           then final vMotion
   - Good for batch     - Good for critical - Best for large VMs
     migration            workloads           over WAN
```

**Replication Assisted vMotion (RAv)** is usually the right choice for production migrations. It pre-syncs data via replication (tolerating WAN latency) then does a final vMotion switchover for minimal downtime.

### Network Extension

HCX can stretch L2 networks across sites so migrated VMs keep their IP addresses:

```
On-Prem                                    Cloud
Segment: 10.1.1.0/24                       Extended: 10.1.1.0/24
  VM-A: 10.1.1.10  ----HCX L2 stretch----  VM-B: 10.1.1.20
                                            (migrated, same IP)
```

**When to use L2 extension:**

- Migration phase -- VMs move in batches, some still on-prem talking to migrated VMs
- Applications with hardcoded IPs that can't be easily changed
- Temporary during cutover

**When to cut L2 extension:**

- After migration completes -- extended L2 adds latency and complexity
- Don't run production permanently on stretched L2 unless you have a specific reason
- Re-IP workloads and move to native cloud networking when practical

## Connectivity Patterns

### VPN (Baseline)

IPsec VPN between on-prem edge and cloud.

```
On-prem firewall/router  ---- IPsec VPN (internet) ----  Cloud VPN gateway
```

- **Bandwidth**: Limited by internet uplink (typically 100Mbps-1Gbps usable)
- **Latency**: Variable, internet-dependent
- **Cost**: Low (existing internet circuit)
- **When to use**: Dev/test, low-bandwidth workloads, initial setup before dedicated circuit

### Dedicated Connectivity

| Cloud  | Service            | What It Is                                   |
| ------ | ------------------ | -------------------------------------------- |
| AWS    | Direct Connect     | Dedicated fiber from colo/on-prem to AWS     |
| Azure  | ExpressRoute       | Dedicated circuit from colo/on-prem to Azure |
| Google | Cloud Interconnect | Dedicated or partner interconnect to Google  |

**Typical specs:**

- 1Gbps, 10Gbps, or 100Gbps circuits
- Predictable latency (not internet-dependent)
- Private traffic (doesn't traverse public internet)
- Required for production hybrid workloads

### Network Design: Hub-and-Spoke

```
                    On-Prem Datacenter
                    (Hub)
                         |
              +----------+----------+
              |          |          |
         Direct      Express    Cloud
         Connect     Route      Interconnect
              |          |          |
           AWS VPC    Azure VNet  GCP VPC
           (Spoke)    (Spoke)     (Spoke)
```

On-prem is the hub. Cloud environments are spokes. All cross-cloud traffic routes through on-prem (or a transit gateway if you need cloud-to-cloud without hairpinning through on-prem).

### Network Design: Cloud Transit

```
         On-Prem                   Transit Gateway / vWAN
              |                           |
              +-- Direct Connect ---------+
                                          |
                                    +-----+-----+
                                    |           |
                                 VPC/VNet    VPC/VNet
                                 Prod        Dev
```

AWS Transit Gateway or Azure Virtual WAN acts as a cloud-side hub. On-prem connects once to the transit layer, which routes to all cloud VPCs/VNets. Scales better than individual peering.

## Hybrid Architecture Patterns

### Pattern 1: DR to Cloud

On-prem is primary. Cloud is the DR target. No production workloads in cloud during normal operation.

```
On-Prem (active)                    Cloud (passive DR)
  Production VMs                      Replica VMs (powered off)
       |                                   |
       +--- SRM + vSphere Replication -----+
            or HCX replication
```

- **Cost model**: Pay for cloud hosts only when DR is active (some providers allow scale-to-zero)
- **RPO**: 5 minutes to hours depending on replication method
- **RTO**: Minutes to hours depending on orchestration
- **Advantage**: No second datacenter to build and operate

### Pattern 2: Cloud Burst

On-prem handles baseline capacity. Cloud absorbs spikes.

```
Normal operation:
  On-prem: 100% of workload

Peak period:
  On-prem: 70% of workload
  Cloud:   30% of workload (burst VMs)
```

- **Use case**: Seasonal retail, batch processing, dev/test surge
- **Mechanism**: HCX migration or pre-staged cloud VMs activated during peaks
- **Challenge**: Data locality. If the database is on-prem, burst compute in cloud has WAN latency to data.
- **Fix**: Either replicate data to cloud or only burst stateless workloads

### Pattern 3: Migration Staging

Use cloud as a temporary landing zone during datacenter migration or hardware refresh.

```
Phase 1: On-prem (old DC)  →  Cloud (temporary)
Phase 2: Cloud (temporary) →  On-prem (new DC)

Or:

Phase 1: On-prem (old DC)  →  Cloud (temporary)
Phase 2: Refactor to cloud-native (permanent)
```

- **HCX L2 extension** keeps IPs stable during migration
- **Application dependency mapping** determines migration wave order
- **Typical wave approach**: Infrastructure first (AD, DNS, DHCP), then applications in dependency order

### Pattern 4: Distributed Workloads

Different workload tiers run where they fit best permanently.

```
On-Prem                              Cloud
  Databases (latency, compliance)      Web tier (global distribution)
  Legacy apps (can't re-architect)     AI/ML (GPU, managed services)
  Regulated data (sovereignty)         Dev/test (elastic capacity)
```

- **Challenge**: Operational complexity. Two platforms to manage, monitor, patch.
- **Mitigation**: Unified management (Aria, vCenter hybrid linked mode), consistent networking (NSX across both)

### Pattern 5: Cloud-Adjacent Services

Keep VMs on-prem but consume cloud services (databases, AI/ML, analytics) over a dedicated link.

```
On-Prem VMs
     |
     +-- Direct Connect / ExpressRoute --+
                                         |
                                    Cloud Services
                                    (RDS, S3, SageMaker,
                                     Azure SQL, Cosmos DB)
```

- **Advantage**: No VM migration. Keep existing infrastructure. Add cloud capabilities incrementally.
- **Use case**: Analytics on cloud data lake, ML model training in cloud, cloud-managed database for new apps
- **Prerequisite**: Dedicated connectivity with sufficient bandwidth

## Cost Considerations

### VMware Cloud Pricing Model

VMware cloud offerings (VMC, AVS, GCVE) are priced per-host-hour. You're paying for dedicated bare-metal hosts, not individual VMs.

```
Cost = Hosts × Per-host hourly rate × Hours

Example (illustrative):
  3 hosts × $9/hr × 730 hrs/month = ~$19,710/month
```

**This is significantly more expensive than running VMs on generic cloud instances.** You're paying for the VMware stack on dedicated hardware. The value proposition is operational consistency, not cost savings.

### When Cloud Hybrid Saves Money

| Scenario           | Why It's Cheaper                                        |
| ------------------ | ------------------------------------------------------- |
| DR site            | Pay for 2-3 hosts vs building a second datacenter       |
| Temporary capacity | Pay only during burst vs buying hardware for peak       |
| Datacenter exit    | Avoid hardware refresh investment when planning to exit |
| Time-to-value      | Deploy in hours vs months of procurement                |

### When Cloud Hybrid Costs More

| Scenario                | Why It's More Expensive                                            |
| ----------------------- | ------------------------------------------------------------------ |
| Steady-state production | Per-host cloud pricing vs amortized on-prem hardware               |
| Large footprint         | Cloud costs scale linearly; on-prem has economies of scale         |
| Data egress heavy       | Pulling data out of cloud incurs egress charges                    |
| Long-term commitment    | 3-year RI on-prem equivalent is cheaper than 3-year cloud reserved |

### Cost Optimization

- **Reserved instances**: 1-year or 3-year commitments for 30-50% savings on cloud hosts
- **Right-size the SDDC**: Don't over-provision cloud hosts. Scale up only when needed.
- **Minimize egress**: Keep data processing close to data storage. Don't push large datasets across the WAN.
- **Evaluate per-workload**: Some workloads belong in cloud, others on-prem. Don't lift-and-shift everything.

## Common Mistakes

| Mistake                                   | Consequence                                          | Fix                                                               |
| ----------------------------------------- | ---------------------------------------------------- | ----------------------------------------------------------------- |
| Treating cloud as just another datacenter | Miss cloud-native advantages, overpay for IaaS       | Evaluate each workload: migrate as-is vs refactor vs cloud-native |
| Permanent L2 stretch in production        | Added latency, failure domain spanning sites         | Re-IP after migration, use L2 stretch only during transition      |
| No dedicated connectivity for production  | Unreliable performance over internet VPN             | Direct Connect / ExpressRoute for production hybrid               |
| Ignoring data gravity                     | Compute in cloud, data on-prem = high latency        | Put compute near data, or replicate data to cloud                 |
| Lift-and-shift everything                 | Cloud costs more than on-prem for steady-state VMs   | Evaluate per workload. Some should stay on-prem.                  |
| No egress cost modeling                   | Surprise cloud bill from data transfer               | Model egress costs before migrating data-heavy workloads          |
| Over-provisioning cloud SDDC              | Paying for idle hosts at cloud prices                | Start small, scale up. Cloud flexibility is the point.            |
| Hybrid without unified monitoring         | Two separate operational views, slow troubleshooting | Aria Operations or equivalent spanning both environments          |
