# vSphere Hardening and Security

## Security Model Overview

vSphere security operates at multiple layers. Each layer has its own attack surface and hardening controls:

```
+--------------------------------------------------+
|  Management Plane                                |
|  vCenter, SDDC Manager, NSX Manager, Aria        |
+--------------------------------------------------+
|  Hypervisor Layer                                |
|  ESXi hosts, VMkernel, hostd, vpxa               |
+--------------------------------------------------+
|  Virtual Network                                 |
|  vSwitches, distributed port groups, NSX segments |
+--------------------------------------------------+
|  Virtual Machines                                |
|  VM configuration, VMware Tools, virtual hardware |
+--------------------------------------------------+
|  Storage                                         |
|  vSAN encryption, datastore access, VMDK files    |
+--------------------------------------------------+
```

Hardening one layer while ignoring others creates a false sense of security. A compromised vCenter with admin credentials bypasses every ESXi-level control.

## ESXi Host Hardening

### Lockdown Mode

Lockdown mode restricts direct access to ESXi hosts, forcing all management through vCenter.

| Mode     | Direct Access           | vCenter Required         | Use Case                       |
| -------- | ----------------------- | ------------------------ | ------------------------------ |
| Disabled | Full (SSH, DCUI, API)   | No                       | Initial setup, troubleshooting |
| Normal   | Exception users only    | Yes                      | Standard production            |
| Strict   | No direct access at all | Yes (even DCUI disabled) | High-security environments     |

**Normal lockdown** is the right choice for most environments. It allows vCenter-managed operations while blocking casual SSH access, but keeps the DCUI available for emergency access if vCenter is down.

**Exception users**: Accounts explicitly allowed direct access even in lockdown mode. Keep this list minimal -- typically just a break-glass emergency account.

**Enabling lockdown:**

```
vCenter > Host > Configure > System > Security Profile > Lockdown Mode
```

### SSH and Shell Access

| Setting                | Recommendation        | Why                                                            |
| ---------------------- | --------------------- | -------------------------------------------------------------- |
| SSH service            | Stopped, manual start | Reduces attack surface. Start only when needed.                |
| ESXi Shell             | Stopped, manual start | Same as SSH. DCUI access is sufficient for emergencies.        |
| Shell timeout          | 900 seconds (15 min)  | Auto-disconnect idle sessions                                  |
| Suppress shell warning | No                    | Keep the warning -- it reminds admins to disable SSH when done |

**Do not leave SSH permanently enabled in production.** Every open service is an attack vector. If you need persistent remote access for automation, use the vSphere API through vCenter instead.

### Account Security

**Root password:**

- Set a strong, unique root password on each host
- Rotate on a defined schedule
- Store in a privileged access management (PAM) vault
- Do not use the same root password across all hosts

**Active Directory integration:**

- Join ESXi hosts to AD for centralized authentication
- Map AD groups to ESXi roles (Admin, ReadOnly)
- Removes dependency on local accounts for day-to-day access
- Root account becomes break-glass only

**Account lockout:**

```
Security.AccountLockFailures = 5    # Lock after 5 failures
Security.AccountUnlockTime = 900    # Unlock after 15 minutes
```

### Certificate Management

vSphere uses TLS certificates throughout the stack. Default behavior uses VMCA (VMware Certificate Authority) as an intermediate CA.

| Model                  | How It Works                                          | When to Use                    |
| ---------------------- | ----------------------------------------------------- | ------------------------------ |
| VMCA as CA             | VMCA issues all certs. Self-signed root.              | Small environments, labs       |
| VMCA as subordinate CA | Enterprise CA signs VMCA. VMCA issues endpoint certs. | Most production environments   |
| Custom certificates    | Enterprise CA issues all certs directly. No VMCA.     | Strict compliance, PKI mandate |

**Recommendation**: VMCA as subordinate CA. It balances automation (VMCA handles cert lifecycle) with trust (enterprise CA root in the chain).

**Certificate rotation**: VMCA certs default to 2-year validity. Plan for renewal before expiry -- expired certs cause service outages across the entire vSphere stack.

### Firewall

ESXi has a built-in host firewall. Default rules allow management traffic and block everything else.

**Review and tighten:**

```bash
# List firewall rules
esxcli network firewall ruleset list

# Restrict a service to specific IP ranges
esxcli network firewall ruleset set -r sshServer -e true
esxcli network firewall ruleset allowedip add -r sshServer -i 10.0.1.0/24
```

**Key principle**: Restrict management services (SSH, vSphere API, SNMP) to management network subnets. Don't allow management traffic from VM networks.

### Host Profile / Config Drift

Use vSphere Host Profiles or vLCM desired state to enforce consistent configuration across hosts:

- NTP settings
- Syslog targets
- Firewall rules
- Advanced settings
- Network configuration

Drift detection alerts when a host deviates from the baseline. Remediation brings it back into compliance.

## vCenter Hardening

### Access Control

**Role-based access (RBAC):**

- Assign minimum necessary privileges
- Use custom roles rather than Administrator for most users
- Avoid assigning permissions at the vCenter root -- scope to datacenters or clusters
- Audit permission assignments regularly

**Common role segmentation:**

| Role            | Scope                   | Typical User                                        |
| --------------- | ----------------------- | --------------------------------------------------- |
| Administrator   | vCenter root            | Infrastructure team lead (1-2 people)               |
| VM Power User   | Specific resource pools | Application teams (power on/off, console, snapshot) |
| Read-Only       | Datacenter              | Monitoring, audit, helpdesk                         |
| Network Admin   | dvSwitch / NSX          | Network team                                        |
| Datastore Admin | Specific datastores     | Storage team                                        |

**SSO and identity:**

- vCenter SSO integrates with Active Directory or LDAP
- Use AD groups for permission assignment (not individual accounts)
- Enable MFA for vCenter access where possible (RSA, SAML IdP)
- The vsphere.local administrator account is break-glass only -- don't use it for daily operations

### Audit Logging

**vCenter events and tasks:**

- Logged by default to the vCenter database
- Retention depends on database size -- configure adequate retention (90+ days)
- Forward to external SIEM (Splunk, Elastic, etc.) for long-term retention and correlation

**Syslog forwarding from ESXi:**

```
# Configure syslog target on each host
esxcli system syslog config set --loghost=tcp://siem.example.com:514

# Or via Host Profile for consistency across all hosts
```

**What to monitor:**

| Event                           | Why It Matters                               |
| ------------------------------- | -------------------------------------------- |
| Permission changes              | Privilege escalation detection               |
| VM power state changes          | Unauthorized activity                        |
| VM creation/deletion            | Asset tracking, rogue VMs                    |
| Host added/removed from cluster | Infrastructure changes                       |
| Login failures                  | Brute force detection                        |
| Configuration changes           | Drift detection                              |
| Snapshot creation               | Snapshot sprawl, potential data exfiltration |

### vCenter Appliance Hardening

The VCSA (vCenter Server Appliance) is a Linux VM. Standard Linux hardening applies:

- Restrict VAMI (management interface) access to management network
- Disable unused services
- Apply VCSA patches promptly (vCenter is a high-value target)
- Back up vCenter configuration regularly (file-based backup)
- Monitor disk space -- vCenter logs can fill the disk and crash services

## Virtual Machine Security

### VM Configuration Hardening

**Disable unnecessary virtual hardware:**

| Setting                            | Value             | Why                                       |
| ---------------------------------- | ----------------- | ----------------------------------------- |
| isolation.tools.copy.disable       | TRUE              | Prevent clipboard copy from VM to client  |
| isolation.tools.paste.disable      | TRUE              | Prevent clipboard paste from client to VM |
| isolation.tools.diskShrink.disable | TRUE              | Prevent guest-initiated disk shrink       |
| isolation.tools.diskWipe.disable   | TRUE              | Prevent guest-initiated disk wipe         |
| mks.enable3d                       | FALSE (if unused) | Reduce attack surface of 3D rendering     |
| RemoteDisplay.maxConnections       | 1                 | Limit concurrent console connections      |

**Remove unnecessary hardware:**

- Floppy drives (default in some templates)
- Serial/parallel ports
- USB controllers (if not needed)
- CD/DVD drives (disconnect after install)

Each virtual hardware device is code in the hypervisor that processes guest input -- minimizing devices minimizes attack surface.

### VMware Tools

- Keep VMware Tools updated -- it's a privileged agent inside the guest
- Use the open-vm-tools package on Linux (maintained by the distro, patched faster)
- Disable Tools features you don't use (drag-and-drop, shared folders in Workstation/Fusion don't apply to ESXi, but clipboard operations do)

### VM Encryption

vSphere VM Encryption encrypts VMDK files, VM swap, and snapshots at rest using a Key Management Server (KMS).

**Architecture:**

```
vCenter ←→ KMS (KMIP-compliant)
  |
  +-- Requests encryption key
  +-- Distributes to ESXi hosts
  +-- ESXi encrypts/decrypts at I/O level
```

**What gets encrypted:**

- VM disk files (.vmdk)
- VM swap file (.vswp)
- Snapshot files

**What does NOT get encrypted:**

- VM configuration file (.vmx) -- contains metadata, not data
- vMotion traffic (use separate vMotion encryption setting)
- VM console traffic (use separate console encryption)

**Performance impact**: 1-5% overhead for AES-NI capable processors (all modern Intel/AMD). Negligible for most workloads.

**vSAN encryption** is a separate feature. It encrypts at the vSAN datastore level rather than per-VM. Choose one approach:

- VM Encryption: Granular (per-VM). Works with any storage.
- vSAN Encryption: Blanket (all data on vSAN). Simpler management. vSAN only.

## Network Security

### Management Network Isolation

**The management network is the highest-value target.** If an attacker reaches vCenter or ESXi management interfaces, they control the entire infrastructure.

```
+-------------------+     +-------------------+     +-------------------+
| Management Network|     | vMotion Network   |     | VM Network        |
| vCenter, ESXi mgmt|     | Host-to-host only |     | Application traffic|
| VLAN 10           |     | VLAN 20           |     | VLAN 100+         |
| Firewalled        |     | No default gateway|     | Per-app segmented |
+-------------------+     +-------------------+     +-------------------+
```

**Rules:**

- Management network on dedicated VLAN, firewalled from VM networks
- vMotion network on dedicated VLAN, no default gateway (host-to-host only)
- vSAN network on dedicated VLAN (if using vSAN)
- No routing between VM networks and management network without explicit firewall rules

### vMotion Encryption

vMotion transfers VM memory in cleartext by default. On shared or untrusted networks, enable encryption:

| Setting       | Behavior                                                  |
| ------------- | --------------------------------------------------------- |
| Disabled      | No encryption (default)                                   |
| Opportunistic | Encrypt if both hosts support it, fall back to cleartext  |
| Required      | Always encrypt. vMotion fails if encryption not possible. |

**Recommendation**: Opportunistic for most environments. Required if vMotion traverses untrusted network segments.

### Distributed Switch Security

Port group security settings control traffic behavior:

| Setting             | Default | Secure Setting | Why                                         |
| ------------------- | ------- | -------------- | ------------------------------------------- |
| Promiscuous mode    | Reject  | Reject         | Prevents VMs from sniffing other VM traffic |
| MAC address changes | Accept  | Reject         | Prevents MAC spoofing                       |
| Forged transmits    | Accept  | Reject         | Prevents source MAC spoofing                |

**Set these to Reject unless a specific application requires otherwise** (some legitimate use cases: nested ESXi, NLB, network monitoring appliances).

## STIG and Compliance Baselines

### What STIGs Are

Security Technical Implementation Guides (STIGs) are DoD-published configuration baselines. VMware publishes vSphere STIGs for:

- ESXi hosts
- vCenter Server
- Virtual machines
- vSAN

Even if you're not DoD, STIGs are a solid hardening checklist.

### Applying STIGs

**Assessment tools:**

- VMware provides PowerCLI scripts for STIG assessment
- vCenter Configuration Profiles (VCF 5+) can enforce STIG settings
- Third-party tools: Puppet, Ansible, Chef with vSphere modules

**Common STIG findings and fixes:**

| STIG Finding                  | Control                        | Fix                                    |
| ----------------------------- | ------------------------------ | -------------------------------------- |
| SSH enabled                   | Disable SSH when not in use    | Stop SSH service, set to manual start  |
| Lockdown mode disabled        | Enable lockdown                | Set Normal lockdown mode               |
| NTP not configured            | Configure time synchronization | Set NTP servers in host config         |
| Syslog not forwarded          | Configure remote syslog        | Set syslog target via esxcli           |
| SNMP community string default | Change or disable SNMP         | Reconfigure or disable SNMP agent      |
| Weak TLS versions             | Disable TLS 1.0/1.1            | Configure minimum TLS 1.2              |
| Default self-signed certs     | Replace with CA-signed         | Use VMCA subordinate or custom certs   |
| VM copy/paste enabled         | Disable clipboard operations   | Set isolation.tools.copy/paste.disable |

### CIS Benchmarks

Center for Internet Security publishes benchmarks for vSphere. Similar scope to STIGs but with scored/not-scored categories. Useful for non-DoD organizations wanting a compliance framework.

## Zero Trust in vSphere

Zero trust in a vSphere context means:

1. **No implicit trust based on network location.** Being on the management VLAN doesn't grant access. Authentication and authorization at every layer.

2. **Microsegmentation for east-west traffic.** NSX distributed firewall enforces per-VM policy regardless of network placement (see [nsx-segmentation.md](nsx-segmentation.md)).

3. **Least privilege for all accounts.** No shared admin accounts. Role-based access scoped to the minimum necessary objects.

4. **Continuous verification.** Audit logging, anomaly detection, configuration drift monitoring. Trust is never assumed, always verified.

5. **Encrypted communications.** vMotion encryption, VM encryption, encrypted syslog, TLS everywhere.

### Zero Trust Implementation Priority

```
1. Segment management plane from workloads        (network isolation)
2. Enforce RBAC with least privilege               (identity)
3. Enable microsegmentation for east-west traffic  (NSX DFW)
4. Encrypt data in transit and at rest             (vMotion, VM encryption)
5. Forward and monitor all logs centrally          (visibility)
6. Automate compliance checking and remediation    (continuous verification)
```

## Common Mistakes

| Mistake                                        | Consequence                                    | Fix                                                     |
| ---------------------------------------------- | ---------------------------------------------- | ------------------------------------------------------- |
| SSH left permanently enabled                   | Persistent attack surface, STIG violation      | Stop SSH, set to manual start, use lockdown mode        |
| Same root password on all hosts                | One compromised host compromises all           | Unique passwords per host, use PAM vault                |
| vCenter admin for daily operations             | Over-privileged access, audit trail is useless | Custom roles with minimum privilege                     |
| Management network accessible from VM networks | Attacker pivots from compromised VM to vCenter | Firewall management network, separate VLANs             |
| Promiscuous mode enabled on port groups        | VMs can sniff other VM traffic                 | Reject unless specifically required                     |
| No syslog forwarding                           | No audit trail for forensics                   | Forward to SIEM, retain 90+ days                        |
| Expired certificates ignored                   | Service outages, trust chain broken            | Monitor cert expiry, automate renewal                   |
| VM encryption without KMS backup               | KMS failure = permanent data loss              | Back up KMS, test key recovery                          |
| STIG compliance treated as one-time            | Configuration drifts back to insecure state    | Continuous compliance monitoring, automated remediation |
