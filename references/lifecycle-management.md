# Lifecycle Management

## Why Lifecycle Management Matters

Infrastructure that isn't patched is infrastructure that's vulnerable. But patching a running environment without disrupting workloads requires planning, tooling, and discipline. vSphere provides two lifecycle management models -- choose based on your environment.

## vSphere Lifecycle Manager (vLCM)

### Two Models: Baselines vs Images

| Model                      | How It Works                                                                        | Best For                                                      |
| -------------------------- | ----------------------------------------------------------------------------------- | ------------------------------------------------------------- |
| **Baselines** (legacy)     | Attach patch bundles to hosts, remediate individually                               | Mixed hardware, incremental patching, brownfield environments |
| **Images** (desired state) | Define a complete host image (ESXi + drivers + firmware). All hosts converge to it. | Homogeneous clusters, VCF, greenfield deployments             |

**vLCM Images are the future.** VMware is investing in image-based management. Baselines still work but won't get new features.

### Image-Based Lifecycle (Recommended)

A vLCM image defines the complete, desired state of every host in a cluster:

```
vLCM Image
  |
  +-- ESXi base image (e.g., ESXi 8.0 Update 3)
  +-- Vendor add-on (e.g., Dell custom image, HPE image)
  +-- Firmware add-on (e.g., Dell OMIVV firmware bundle)
  +-- Components (individual VIBs: drivers, agents)
```

**How it works:**

1. Define the image at the cluster level
2. vLCM compares each host's current state to the desired image
3. Non-compliant hosts show as "drifted"
4. Remediation brings hosts into compliance (rolling reboot)

```
Cluster: Production-01
  Image: ESXi 8.0U3 + Dell 8.0.300 + Firmware 24.10
  |
  +-- Host 1: Compliant ✓
  +-- Host 2: Compliant ✓
  +-- Host 3: Non-compliant (missing firmware update) ✗
  +-- Host 4: Compliant ✓
```

### Baseline-Based Lifecycle (Legacy)

Baselines are collections of patches attached to hosts or clusters:

| Baseline Type | Contains                           | Use Case                |
| ------------- | ---------------------------------- | ----------------------- |
| Patch         | Individual ESXi patches            | Targeted security fixes |
| Extension     | Third-party VIBs (agents, drivers) | Adding host software    |
| Upgrade       | Full ESXi version                  | Major version upgrades  |

**Workflow:**

1. Import patches into vLCM depot (or sync from VMware online depot)
2. Create baselines from available patches
3. Attach baselines to clusters or hosts
4. Scan for compliance
5. Remediate non-compliant hosts

Baselines give finer-grained control (pick individual patches) but don't manage firmware and create more drift risk since hosts can end up at different patch levels.

## Firmware Management

### Hardware Support Managers (HSM)

vLCM can manage firmware through vendor plugins:

| Vendor | Plugin                                            | What It Manages                                      |
| ------ | ------------------------------------------------- | ---------------------------------------------------- |
| Dell   | OpenManage Integration for VMware vCenter (OMIVV) | BIOS, iDRAC, NIC, storage controller, drive firmware |
| HPE    | OneView for VMware vCenter                        | iLO, BIOS, NIC, storage firmware                     |
| Lenovo | XClarity Integrator                               | IMM/XCC, BIOS, NIC firmware                          |

**With HSM + vLCM Images:**

```
vLCM Image = ESXi base + vendor add-on + firmware bundle
                                            |
                                     HSM applies firmware
                                     during remediation
```

Single remediation operation updates ESXi patches AND firmware. Host reboots once instead of separately for each.

**Without HSM:**
Firmware updates are a separate process (vendor tools, manual, or out-of-band). More operational overhead, more maintenance windows.

### Firmware Update Sequence

When vLCM remediates a host with both ESXi and firmware updates:

```
1. Host enters maintenance mode (DRS evacuates VMs)
2. Firmware staged to BMC/iDRAC/iLO
3. ESXi patches applied
4. Host reboots
5. Firmware applied during POST (before ESXi loads)
6. Host boots with updated ESXi + firmware
7. Host exits maintenance mode (DRS returns VMs)
```

The host reboots once. Firmware that requires a separate reboot cycle (rare) will trigger an additional reboot automatically.

## Upgrade Sequencing

### The Upgrade Order

vSphere component upgrades must follow a specific sequence. Upgrading out of order can break compatibility.

```
Step 1: vCenter Server
         |
Step 2: ESXi hosts (cluster by cluster)
         |
Step 3: VMware Tools (VM by VM or in bulk)
         |
Step 4: VM hardware version (VM by VM, requires reboot)
```

**Always upgrade vCenter first.** vCenter N supports ESXi N and N-1. ESXi N does not support vCenter N-1. If you upgrade hosts before vCenter, vCenter can't manage them.

### VCF Upgrade Sequence

In a VCF environment, SDDC Manager orchestrates the full stack:

```
Step 1: SDDC Manager itself
         |
Step 2: vCenter Server
         |
Step 3: NSX Manager cluster
         |
Step 4: ESXi hosts (management domain first, then workload domains)
         |
Step 5: vSAN (on-disk format, if applicable)
         |
Step 6: Aria suite components
```

**SDDC Manager enforces this order.** It won't let you upgrade components out of sequence. It also pre-checks compatibility before starting.

### NSX Upgrade Considerations

NSX upgrades are multi-step and affect the network data plane:

```
1. NSX Manager cluster (rolling upgrade, 1 node at a time)
2. NSX Edge nodes (rolling, traffic fails over between edges)
3. Host transport nodes (rolling, VMs maintain connectivity via redundant TEPs)
```

**Key risks:**

- Upgrading NSX Edges disrupts stateful services (NAT, VPN, load balancer) during failover
- Host transport node upgrade requires ESXi maintenance mode
- Always upgrade NSX Manager before Edges and transport nodes

### Compatibility Matrix

Before any upgrade, check the VMware Interoperability Matrix:

| Check                                     | Why                                          |
| ----------------------------------------- | -------------------------------------------- |
| vCenter ↔ ESXi compatibility              | vCenter must support the ESXi version        |
| NSX ↔ vSphere compatibility               | NSX version must support the vSphere version |
| vSAN ↔ ESXi compatibility                 | vSAN features tied to ESXi version           |
| Third-party products (backup, monitoring) | Vendor support for new vSphere version       |
| Hardware compatibility (HCL)              | Drivers and firmware for new ESXi version    |

**Don't skip HCL checks.** A driver that worked on ESXi 7 may not be included in ESXi 8. Missing drivers mean lost storage or network connectivity on reboot.

## Maintenance Mode and Remediation

### Host Remediation Workflow

```
1. Pre-check: vLCM verifies cluster can tolerate host removal
     - DRS can evacuate all VMs
     - HA admission control allows temporary capacity loss
     - vSAN has enough capacity for data migration/accessibility
         |
2. Enter maintenance mode
     - DRS migrates VMs to other hosts
     - vSAN: ensure accessibility (default) or full data migration
         |
3. Apply updates
     - ESXi patches installed
     - Firmware staged (if HSM configured)
         |
4. Reboot
     - Firmware applied during POST
     - ESXi loads with new patches
         |
5. Exit maintenance mode
     - Host rejoins cluster
     - DRS rebalances VMs back
     - vSAN resynchronizes if needed
         |
6. Next host (sequential by default)
```

### Parallel Remediation

vLCM can remediate multiple hosts simultaneously if the cluster has enough capacity:

```
Cluster: 8 hosts
  Parallel remediation: 2 hosts at a time
  Effective capacity during remediation: 6 hosts (25% reduction)
```

**Prerequisites for parallel remediation:**

- HA admission control must tolerate the reduced host count
- DRS must have enough capacity to place all VMs
- vSAN must tolerate multiple hosts in maintenance simultaneously

**Risk**: More hosts out simultaneously means less capacity headroom. If another host fails during remediation, you may not have enough capacity. Conservative environments remediate one host at a time.

### Maintenance Mode Failures

Common reasons maintenance mode gets stuck:

| Symptom                         | Cause                                                  | Fix                                             |
| ------------------------------- | ------------------------------------------------------ | ----------------------------------------------- |
| VM won't migrate                | DRS anti-affinity rule prevents placement              | Temporarily relax rule or manually migrate      |
| VM won't migrate                | VM has local storage (CD/ISO mounted, local datastore) | Disconnect ISO, storage vMotion first           |
| VM won't migrate                | Insufficient resources on target hosts                 | Power off low-priority VMs to free capacity     |
| vSAN data migration takes hours | Full data migration selected on large host             | Use "ensure accessibility" for routine patching |
| Host stuck entering maintenance | VM with USB passthrough can't vMotion                  | Remove USB passthrough, migrate, reattach       |

## Patch Management Strategy

### Cadence

| Environment        | Patch Frequency                     | Approach                                                       |
| ------------------ | ----------------------------------- | -------------------------------------------------------------- |
| Production         | Quarterly or after critical CVEs    | Test in lower environments first, scheduled maintenance window |
| Dev/Test           | Monthly                             | More aggressive, acceptable risk of issues                     |
| Management cluster | Quarterly (aligned with production) | Patch management cluster slightly before or after production   |

### Staging Environments

```
1. Lab / Dev cluster     → Patch immediately, validate functionality
2. Pre-production cluster → Patch 1-2 weeks after lab validation
3. Production cluster     → Patch after pre-prod stabilization
```

**Never patch production first.** Even VMware-released patches occasionally cause issues. Let lower environments surface problems.

### Emergency Patching (Critical CVEs)

When a critical vulnerability drops (e.g., CVE with active exploitation):

1. Assess exposure: Is the vulnerable component reachable? Is there a workaround?
2. Test patch in lab: Even under pressure, validate in a non-production environment
3. Shorten the staging pipeline: Lab validation hours, not weeks
4. Patch production: Schedule emergency window or accept the risk of unplanned remediation
5. Document: Record the exception to normal cadence for audit

### Rollback Planning

**Before patching, always have a rollback plan:**

- ESXi patches: Boot from previous image using alt-boot bank (`/altbootbank`)
- vCenter: File-based backup before upgrade (restore to snapshot or backup)
- NSX: Snapshot NSX Manager VMs before upgrade
- Firmware: Most firmware updates are not easily rolled back -- test thoroughly

**ESXi alt-boot bank:**

```bash
# Check boot bank status
bootbank=$(esxcli system boot device get)

# After a failed patch, reboot to previous image
esxcli system boot device set --device=/altbootbank

# Or at boot menu, select "Rollback" option
```

## VMware Tools and VM Hardware

### VMware Tools Upgrades

VMware Tools can be upgraded independently of ESXi. Options:

| Method                                   | Disruption                           | When to Use                            |
| ---------------------------------------- | ------------------------------------ | -------------------------------------- |
| Manual (per VM)                          | VM reboot usually required (Windows) | Small environments, controlled rollout |
| Automatic at power cycle                 | Upgrades when VM reboots             | Set-and-forget for non-critical VMs    |
| Bulk via vCenter                         | Schedule across many VMs             | Large environments                     |
| Guest OS package manager (open-vm-tools) | No reboot on Linux                   | Linux VMs                              |

**Recommendation**: Use open-vm-tools on Linux (updated via yum/apt with OS patches). Use vCenter-managed upgrades on Windows with scheduled reboots.

### VM Hardware Version

VM hardware version determines which virtual hardware features are available. Upgrading requires a VM reboot.

**Key decision**: You don't have to upgrade VM hardware version with every ESXi upgrade. Only upgrade when you need features from the new hardware version.

| When to Upgrade                                                           | When to Skip                                            |
| ------------------------------------------------------------------------- | ------------------------------------------------------- |
| Need new virtual hardware features (NVMe controller, newer vGPU profiles) | VM is stable, no new feature requirements               |
| Retiring support for old ESXi versions                                    | Need to maintain backward compatibility with older ESXi |
| Template refresh cycle                                                    | Hundreds of VMs with no downtime window                 |

**VM hardware version is one-way.** You can't downgrade. This means the VM can't vMotion to hosts running older ESXi versions that don't support that hardware version.

## Common Mistakes

| Mistake                                       | Consequence                                        | Fix                                                           |
| --------------------------------------------- | -------------------------------------------------- | ------------------------------------------------------------- |
| Upgrading ESXi before vCenter                 | vCenter can't manage upgraded hosts                | Always upgrade vCenter first                                  |
| Skipping HCL check                            | Missing drivers after reboot, lost storage/network | Check VMware HCL before every upgrade                         |
| Patching production first                     | Discover issues in production                      | Stage through lab → pre-prod → prod                           |
| No vCenter backup before upgrade              | Failed upgrade with no rollback path               | File-based backup before every vCenter upgrade                |
| Full data migration for routine vSAN patching | Hours-long maintenance windows                     | Use "ensure accessibility" unless host is leaving permanently |
| Ignoring VMware Tools on Linux                | Stale guest agent, missing features                | Use open-vm-tools, update with OS patches                     |
| Upgrading VM hardware version unnecessarily   | Loses backward compatibility for no benefit        | Only upgrade when you need new features                       |
| No rollback plan                              | Failed patch with no recovery path                 | Document rollback steps before every maintenance window       |
| Patching all clusters simultaneously          | Environment-wide outage if patch causes issues     | Stagger across clusters, validate between each                |
