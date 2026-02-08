# Monitoring and Observability

## Philosophy

Monitoring tells you something is wrong. Observability tells you why. A mature vSphere environment needs both:

- **Monitoring**: Thresholds, alerts, dashboards. "CPU ready is above 5%."
- **Observability**: Correlation, drill-down, root cause. "CPU ready is high because a 16-vCPU VM is causing co-stop on an overcommitted NUMA node."

## esxtop Field Reference

esxtop is the ground truth for ESXi performance. Everything else (vCenter, Aria) is derived from the same counters but sampled less frequently.

### CPU View (default view)

Press `c` for CPU view.

| Field  | Meaning                                       | Healthy           | Warning | Critical |
| ------ | --------------------------------------------- | ----------------- | ------- | -------- |
| %USED  | CPU time consumed by the world                | Context dependent | --      | --       |
| %RDY   | Time waiting for a physical CPU               | < 5%              | 5-10%   | > 10%    |
| %CSTP  | Time halted waiting for sibling vCPUs to sync | < 3%              | 3-5%    | > 5%     |
| %MLMTD | Time throttled by CPU limit                   | 0%                | Any     | --       |
| %SWPWT | Time waiting on swapped memory                | 0%                | Any     | --       |
| %WAIT  | Time in idle or I/O wait                      | Varies            | --      | --       |
| %SYS   | Time in VMkernel system calls                 | < 5%              | 5-10%   | > 10%    |
| NWLD   | Number of worlds (threads)                    | --                | --      | --       |
| GID    | Group ID (VM identifier)                      | --                | --      | --       |

**Key formula:**

```
True CPU Demand = %USED + %RDY + %CSTP
```

If demand >> %USED, the VM is constrained. %RDY means host overcommit. %CSTP means too many vCPUs for the workload.

### Memory View

Press `m` for memory view.

| Field   | Meaning                        | Healthy        | Warning             | Critical                |
| ------- | ------------------------------ | -------------- | ------------------- | ----------------------- |
| MEMSZ   | Configured VM memory (MB)      | --             | --                  | --                      |
| GRANT   | Physical memory granted (MB)   | Close to MEMSZ | --                  | --                      |
| CONSUMD | Physical memory consumed (MB)  | --             | --                  | --                      |
| ACTV    | Actively used memory (MB)      | --             | --                  | --                      |
| MCTLSZ  | Balloon driver target (MB)     | 0              | Any > 0             | Large relative to MEMSZ |
| MCTL?   | Balloon driver active (Y/N)    | N              | Y                   | --                      |
| SWCUR   | Current swap usage (MB)        | 0              | Any > 0             | > 1% of MEMSZ           |
| SWTGT   | Swap target (MB)               | 0              | Any > 0             | --                      |
| ZIP/s   | Compression pages per second   | 0              | Any > 0             | --                      |
| UNZIP/s | Decompression pages per second | 0              | Any > 0             | --                      |
| CACHESZ | Compression cache size (MB)    | 0              | --                  | --                      |
| N%L     | % memory local to NUMA node    | > 95%          | 80-95%              | < 80%                   |
| NRMEM   | Remote NUMA memory (MB)        | Near 0         | --                  | High relative to total  |
| NHN     | NUMA home node                 | Stable         | Frequently changing | --                      |

**Memory reclamation escalation visible in esxtop:**

```
Healthy:     MCTLSZ=0, SWCUR=0, ZIP/s=0
Ballooning:  MCTLSZ > 0, SWCUR=0
Compressing: ZIP/s > 0, SWCUR=0
Swapping:    SWCUR > 0  ← performance degradation is real at this point
```

### Disk View

Press `d` for disk (virtual) or `u` for storage adapter view.

| Field    | Meaning                     | Healthy                    | Warning       | Critical     |
| -------- | --------------------------- | -------------------------- | ------------- | ------------ |
| GAVG     | Guest-observed latency (ms) | < 10ms (SSD), < 20ms (HDD) | 10-20ms (SSD) | > 20ms (SSD) |
| KAVG     | VMkernel latency (ms)       | < 2ms                      | 2-5ms         | > 5ms        |
| DAVG     | Device latency (ms)         | < 5ms (SSD), < 1ms (NVMe)  | 5-10ms (SSD)  | > 10ms (SSD) |
| QAVG     | Queue latency (ms)          | < 1ms                      | 1-5ms         | > 5ms        |
| CMDS/s   | Commands per second (IOPS)  | Context dependent          | --            | --           |
| READS/s  | Read IOPS                   | --                         | --            | --           |
| WRITES/s | Write IOPS                  | --                         | --            | --           |
| MBREAD/s | Read throughput (MB/s)      | --                         | --            | --           |
| MBWRTN/s | Write throughput (MB/s)     | --                         | --            | --           |
| APTS/s   | Aborts per second           | 0                          | Any > 0       | --           |
| RESETS/s | Resets per second           | 0                          | Any > 0       | --           |

**Latency decomposition:**

```
GAVG = KAVG + DAVG

KAVG high, DAVG low → VMkernel bottleneck (queuing, scheduler)
DAVG high, KAVG low → Storage device bottleneck (disk, array, vSAN)
Both high            → Storage subsystem saturated
QAVG high            → Queue depth exceeded, I/O waiting
```

### Network View

Press `n` for network view.

| Field   | Meaning                        | Healthy             | Warning  |
| ------- | ------------------------------ | ------------------- | -------- |
| MbTX/s  | Transmit throughput (Mbps)     | < 70% of link speed | > 70%    |
| MbRX/s  | Receive throughput (Mbps)      | < 70% of link speed | > 70%    |
| PKTTX/s | Packets transmitted per second | --                  | --       |
| PKTRX/s | Packets received per second    | --                  | --       |
| DRPTX/s | Dropped TX packets per second  | 0                   | Any > 0  |
| DRPRX/s | Dropped RX packets per second  | 0                   | Any > 0  |
| %DRPTX  | Drop percentage TX             | 0%                  | Any > 0% |
| %DRPRX  | Drop percentage RX             | 0%                  | Any > 0% |

Packet drops indicate NIC saturation, misconfigured traffic shaping, or physical switch issues.

### esxtop Batch Mode

For historical capture and offline analysis:

```bash
# Capture all counters, 5-second intervals, 720 samples (1 hour)
esxtop -b -d 5 -n 720 > /tmp/esxtop_capture.csv

# Capture specific counter groups
esxtop -b -d 10 -n 360 -c /path/to/esxtop_config > /tmp/esxtop_capture.csv
```

Import the CSV into Excel, Perfmon (Windows), or a time-series database for analysis.

**Save an esxtop configuration to capture only relevant fields:**

```bash
# In interactive esxtop, configure fields with 'f', then save:
# Press 'W' to write config to ~/.esxtop50rc
```

## vCenter Performance Charts

### Real-Time vs Historical

| Mode       | Sampling Interval | Retention | Use Case                   |
| ---------- | ----------------- | --------- | -------------------------- |
| Real-time  | 20 seconds        | 1 hour    | Active troubleshooting     |
| Past Day   | 5 minutes         | 1 day     | Recent issue investigation |
| Past Week  | 30 minutes        | 1 week    | Trend analysis             |
| Past Month | 2 hours           | 1 month   | Capacity trending          |
| Past Year  | 1 day             | 1 year    | Long-term planning         |

**Implication**: A 30-second CPU spike visible in esxtop will be averaged away in the "Past Week" chart. For transient issues, use real-time charts or esxtop.

### Key vCenter Charts by Troubleshooting Scenario

| Symptom              | Charts to Check                            | What to Look For                          |
| -------------------- | ------------------------------------------ | ----------------------------------------- |
| VM slow              | VM > CPU > Ready, CoStop                   | Ready > 5% or CoStop > 3%                 |
| VM slow              | VM > Memory > Balloon, Swap                | Ballooning or swap activity               |
| VM slow              | VM > Disk > Latency                        | GAVG > 10ms                               |
| Host overloaded      | Host > CPU > Usage, Ready                  | Sustained > 80% usage                     |
| Host memory pressure | Host > Memory > Balloon, Swap, Compression | Any reclamation activity                  |
| Cluster imbalanced   | Cluster > CPU/Memory > Usage per host      | One host significantly higher than others |
| Storage slow         | Datastore > Performance > Latency          | Read or write latency spikes              |
| Network issues       | Host > Network > Packets dropped           | Non-zero drops                            |

## Aria Operations (vROps)

### What It Adds Over vCenter

| Capability               | vCenter                | Aria Operations                              |
| ------------------------ | ---------------------- | -------------------------------------------- |
| Real-time metrics        | Yes (20s)              | Yes (20s or custom)                          |
| Historical data          | Up to 1 year (coarse)  | Years (configurable retention)               |
| Alerting                 | Basic (vCenter alarms) | Adaptive thresholds, anomaly detection       |
| Capacity planning        | Minimal                | What-if, right-sizing, time-remaining        |
| Cross-object correlation | Manual                 | Automatic (link VM issue to host to storage) |
| Custom dashboards        | Limited                | Extensive, shareable                         |
| Compliance               | None                   | STIG, CIS benchmark checking                 |
| Cost analysis            | None                   | Show-back/charge-back by department          |

### Key Dashboards

**Operations Overview:**

- Cluster health (red/yellow/green per cluster)
- Top VMs by CPU ready, memory contention, disk latency
- Hosts with most contention
- Alerts requiring attention

**Capacity Dashboard:**

- Time remaining per cluster (CPU, memory, storage)
- Reclaimable capacity (oversized VMs)
- Growth trend per cluster
- What-if scenarios (add hosts, add VMs)

**Right-Sizing Dashboard:**

- VMs with consistent CPU idle > 75%
- VMs with active memory < 50% of configured
- Recommended vCPU and memory reductions
- Estimated savings

**Performance Dashboard:**

- VMs with degraded performance (adaptive thresholds)
- Storage latency heatmap
- Network throughput by port group
- NUMA efficiency (local vs remote memory ratio)

### Adaptive Thresholds

Unlike static thresholds ("alert if CPU > 80%"), Aria Operations learns each object's normal behavior and alerts on deviations:

```
VM normally runs at 60% CPU.
Static threshold: No alert (below 80%)
Adaptive threshold: Alert if it suddenly runs at 90% (anomaly for this VM)

VM normally runs at 85% CPU (database).
Static threshold: Constant alert (above 80%)
Adaptive threshold: No alert (85% is normal for this VM)
```

This dramatically reduces alert noise. The database that always runs hot stops generating false alerts. The web server that suddenly spikes gets flagged.

## Alerting Strategy

### Alert Tiers

| Tier               | Severity              | Response                 | Example                                                   |
| ------------------ | --------------------- | ------------------------ | --------------------------------------------------------- |
| P1 - Critical      | Service impacting     | Immediate (page on-call) | Host failure, vSAN degraded, vCenter down                 |
| P2 - Warning       | Performance degraded  | Same business day        | High CPU ready, memory ballooning, storage latency spike  |
| P3 - Informational | Trending toward issue | Next maintenance window  | Capacity approaching 70%, certificate expiring in 30 days |
| P4 - Advisory      | Good to know          | Review weekly            | VM snapshot older than 3 days, orphaned VMDKs             |

### Essential Alerts

**P1 -- Critical (page-worthy):**

| Alert                    | Condition                                            | Why                                              |
| ------------------------ | ---------------------------------------------------- | ------------------------------------------------ |
| Host unreachable         | Host connection state = disconnected/not responding  | HA event, potential VM downtime                  |
| vSAN health degraded     | Cluster health = degraded or objects = non-compliant | Data at risk                                     |
| vCenter service failure  | vpxd, vsphere-client, or SSO service down            | Management plane lost                            |
| Datastore space critical | > 90% full                                           | VM power-on failures, snapshot failures          |
| HA failover triggered    | HA restart event                                     | VMs recovered but root cause needs investigation |

**P2 -- Warning (business hours):**

| Alert                | Condition                                | Why                                     |
| -------------------- | ---------------------------------------- | --------------------------------------- |
| CPU ready high       | > 10% sustained for 30 minutes on any VM | VM performance degradation              |
| Memory swap active   | SWCUR > 0 for any VM                     | Severe performance impact               |
| Storage latency high | GAVG > 20ms sustained                    | Application slowness                    |
| vSAN resync active   | Resync objects > 0 for extended period   | Rebuild in progress, performance impact |
| Certificate expiring | < 30 days to expiry                      | Service outage if expired               |

**P3 -- Informational (maintenance window):**

| Alert                      | Condition                             | Why                                        |
| -------------------------- | ------------------------------------- | ------------------------------------------ |
| Capacity approaching limit | Any dimension > 70%                   | Procurement trigger                        |
| VM snapshot age            | Snapshot > 72 hours                   | Snapshot sprawl, storage waste             |
| Host config drift          | Host non-compliant with profile/image | Security and consistency risk              |
| VMware Tools outdated      | Tools version > 1 major behind        | Missing features, guest integration issues |

### Alert Routing

| Tier | Channel                        | Audience                            |
| ---- | ------------------------------ | ----------------------------------- |
| P1   | PagerDuty / Opsgenie / phone   | On-call infrastructure engineer     |
| P2   | Slack / Teams channel + ticket | Infrastructure team                 |
| P3   | Email digest / ticket          | Infrastructure team (weekly review) |
| P4   | Dashboard only                 | Reviewed in capacity meetings       |

## Syslog and Log Management

### What to Collect

| Source  | Log                     | Why                                                |
| ------- | ----------------------- | -------------------------------------------------- |
| ESXi    | /var/log/vmkernel.log   | Kernel events, hardware errors, storage issues     |
| ESXi    | /var/log/hostd.log      | Host management daemon (API calls, task execution) |
| ESXi    | /var/log/vpxa.log       | vCenter agent on host (communication issues)       |
| ESXi    | /var/log/vobd.log       | vSphere Observability daemon (events)              |
| vCenter | vpxd.log                | Core vCenter service (task execution, errors)      |
| vCenter | vsphere-client.log      | UI issues                                          |
| NSX     | syslog from NSX Manager | Firewall rule hits, routing changes                |
| NSX     | DFW flow logs           | East-west traffic visibility                       |

### Syslog Configuration

```bash
# ESXi host syslog target
esxcli system syslog config set --loghost=tcp://syslog.example.com:514

# Verify
esxcli system syslog config get

# Reload syslog
esxcli system syslog reload
```

Configure via Host Profile for consistency across all hosts.

### Log Analysis Use Cases

| Scenario                  | What to Search                               | Where                                     |
| ------------------------- | -------------------------------------------- | ----------------------------------------- |
| VM performance issue      | "scsi" + "abort" or "reset" in vmkernel.log  | Storage timeout causing I/O stalls        |
| Host purple screen (PSOD) | "PF Exception" or "panic" in vmkernel.log    | Hardware fault or driver bug              |
| vMotion failure           | "migrate" in vpxa.log and hostd.log          | Network, compatibility, or resource issue |
| HA isolation              | "isolated" in fdm.log                        | Network partition between hosts           |
| Login investigation       | "logged in" or "Authentication" in hostd.log | Security audit                            |

## Metrics Hierarchy

When troubleshooting, work top-down:

```
1. Cluster level
   Is the cluster healthy? Capacity OK? DRS balanced?
       |
2. Host level
   Which host is the problem? CPU/memory/storage contention?
       |
3. VM level
   Which VM is affected? Ready? CoStop? Balloon? Swap? Disk latency?
       |
4. Guest OS level
   What's happening inside the VM? Process CPU? Disk queue? Memory pressure?
       |
5. Application level
   What does the application say? Slow queries? Connection timeouts?
```

**Don't jump to application-level troubleshooting without ruling out infrastructure causes first.** A "slow database" might be a swapping VM, not a bad query.

## Common Mistakes

| Mistake                                            | Consequence                                                  | Fix                                                          |
| -------------------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| Alerting on allocated instead of consumed          | Constant false alerts for memory                             | Alert on active/consumed metrics                             |
| Static thresholds for everything                   | Alert fatigue, missed anomalies                              | Use adaptive thresholds (Aria Operations)                    |
| No syslog forwarding                               | Logs lost on host reboot or failure                          | Forward to central syslog/SIEM                               |
| Ignoring esxtop for real-time issues               | Miss transient spikes averaged away in vCenter               | Use esxtop for active troubleshooting                        |
| Alerting without routing                           | Alerts go to email nobody reads                              | Route P1 to pager, P2 to Slack, P3 to digest                 |
| Monitoring infrastructure but not management plane | vCenter goes down, nobody notices until VMs can't be managed | Monitor vCenter services and VCSA health                     |
| No baseline comparison                             | Can't distinguish normal from abnormal                       | Establish baselines before setting thresholds                |
| Too many P1 alerts                                 | On-call fatigue, real issues missed                          | Ruthlessly classify. If it doesn't wake you up, it's not P1. |
