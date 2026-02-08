# NSX Segmentation Patterns

## Core Concepts

### Overlay vs Underlay

NSX decouples the logical network from the physical network:

- **Underlay**: Physical switches and routers. Carries encapsulated traffic between ESXi hosts. Only needs IP reachability between TEPs (Tunnel Endpoints) -- no VLAN sprawl.
- **Overlay**: Logical segments built on GENEVE encapsulation. Created and destroyed through software without touching physical infrastructure.

```
VM-A (Host 1)                          VM-B (Host 2)
  |                                      |
  +-- Logical Segment (overlay) ---------+
      |                              |
      +-- GENEVE Tunnel (underlay) --+
          |                      |
        TEP (vmk)              TEP (vmk)
          |                      |
        Physical Switch Fabric (just needs L3)
```

**Why this matters**: Network provisioning goes from "submit a ticket for a VLAN" to "click a button." Segmentation decisions become software policy, not hardware topology.

### Segments

A segment is the NSX equivalent of a VLAN-backed port group -- an L2 broadcast domain. But unlike VLANs:

- No 4096 limit (GENEVE VNI is 24-bit: ~16 million segments)
- Not tied to physical switch configuration
- Can span hosts without trunk port changes
- Subnets attach to tier-1 gateways for routing

### Tier-0 and Tier-1 Gateways

NSX uses a two-tier routing model:

```
                External / Physical Network
                         |
                  +------+------+
                  | Tier-0 GW   |  North-south routing, BGP/OSPF peering
                  +------+------+
                    |    |    |
            +-------+   |   +-------+
            |            |           |
      +-----+----+ +----+-----+ +---+------+
      | Tier-1   | | Tier-1   | | Tier-1   |
      | Prod     | | Dev      | | DMZ      |
      +----+-----+ +----+-----+ +----+-----+
           |             |            |
       Segments      Segments     Segments
```

| Gateway | Role               | Typical Use                                                                                     |
| ------- | ------------------ | ----------------------------------------------------------------------------------------------- |
| Tier-0  | North-south edge   | Connects NSX to physical network. Runs BGP/OSPF. Handles NAT, VPN, edge firewall                |
| Tier-1  | Tenant/zone router | One per environment or tenant. Connects segments. Applies gateway firewall, NAT, load balancing |

**Design rule**: Tier-0 is shared infrastructure. Tier-1 is per-tenant or per-zone. You rarely need more than one tier-0 (unless multi-tenancy requires separate external uplinks).

## Segmentation Models

### Zone-Based Segmentation

Classic network zoning mapped to NSX constructs:

```
Tier-0 (Edge)
  |
  +-- Tier-1: DMZ
  |     +-- Segment: dmz-web (10.1.1.0/24)
  |     +-- Segment: dmz-api (10.1.2.0/24)
  |
  +-- Tier-1: Production
  |     +-- Segment: prod-app (10.2.1.0/24)
  |     +-- Segment: prod-db  (10.2.2.0/24)
  |
  +-- Tier-1: Dev
        +-- Segment: dev-app (10.3.1.0/24)
        +-- Segment: dev-db  (10.3.2.0/24)
```

**Inter-zone traffic** flows through tier-1 to tier-0 and back down to the other tier-1, where gateway firewall rules enforce policy. This is the natural enforcement point for zone-to-zone rules (e.g., "DMZ can reach prod-app on 443 only").

**Intra-zone traffic** stays within the tier-1. Distributed firewall rules enforce east-west policy within a zone (e.g., "web servers can't talk directly to database servers").

### Application-Centric Segmentation

Instead of network zones, segment by application:

```
Tier-0
  |
  +-- Tier-1: App-OrderSystem
  |     +-- Segment: orders-web
  |     +-- Segment: orders-api
  |     +-- Segment: orders-db
  |
  +-- Tier-1: App-Inventory
  |     +-- Segment: inventory-api
  |     +-- Segment: inventory-db
  |
  +-- Tier-1: SharedServices
        +-- Segment: dns-ntp
        +-- Segment: monitoring
        +-- Segment: identity
```

Each application gets its own tier-1, making blast radius containment natural. A compromise in the order system can't laterally move to inventory without crossing a gateway firewall.

### Microsegmentation (Zero Trust)

The most granular model. Policy is applied per-VM or per-workload, regardless of network placement. Two VMs on the same segment can have completely different firewall rules.

This is where the **distributed firewall (DFW)** does the heavy lifting -- not the gateway firewalls.

## Distributed Firewall (DFW)

### How It Works

The DFW runs in the ESXi kernel at the vNIC level. Every packet entering or leaving a VM passes through the firewall -- even traffic between VMs on the same host and segment.

```
VM-A ---[vNIC]---[DFW]---[Segment]---[DFW]---[vNIC]--- VM-B
                   ^                    ^
             Rules enforced        Rules enforced
             at source             at destination
```

**This is the key architectural difference from traditional firewalls**: you don't route traffic through a chokepoint. The firewall is distributed to every workload, eliminating hairpin traffic and single points of failure.

### Rule Processing

DFW rules are processed top-down, first match wins:

```
1. Emergency rules       (highest priority -- block known threats)
2. Infrastructure rules  (DNS, NTP, monitoring, vMotion)
3. Environment rules     (zone-to-zone: prod can't reach dev)
4. Application rules     (app-specific: web→app on 8080, app→db on 3306)
5. Default rule          (deny all -- zero trust baseline)
```

Each category maps to a DFW section, making rule management tractable at scale.

### Security Groups

Rules don't reference IP addresses directly. They reference **groups** -- dynamic collections of VMs defined by:

| Criteria            | Example                               | Use Case                         |
| ------------------- | ------------------------------------- | -------------------------------- |
| VM tag              | `scope: Environment, tag: Production` | Environment segmentation         |
| VM name pattern     | `Name starts with "web-"`             | Convention-based grouping        |
| Segment membership  | `Connected to segment prod-db`        | Network-based grouping           |
| OS type             | `Guest OS = Linux`                    | OS-level policy                  |
| Computed (AD group) | `AD group = DB-Admins`                | Identity-based microsegmentation |

**Why tags over IPs**: VMs get added to the right security group automatically when tagged. No rule updates when VMs are added, removed, or re-IPed.

### Example: Three-Tier App Microsegmentation

Groups:

```
Group: orders-web    → Tag: app=orders, tier=web
Group: orders-api    → Tag: app=orders, tier=api
Group: orders-db     → Tag: app=orders, tier=db
Group: monitoring    → Tag: role=monitoring
```

DFW rules:

```
Section: Orders Application
  Allow  orders-web  → orders-api   TCP 8080    (web calls API)
  Allow  orders-api  → orders-db    TCP 3306    (API queries database)
  Allow  monitoring  → orders-*     TCP 9090    (Prometheus scrape)
  Drop   orders-web  → orders-db    Any         (web can't touch DB directly)
  Drop   Any         → orders-*     Any         (default deny to orders app)
```

Result: even if an attacker compromises a web server, they can't reach the database. They'd have to pivot through the API tier, which has its own authentication.

## Gateway Firewall vs Distributed Firewall

| Aspect              | Gateway Firewall                      | Distributed Firewall                  |
| ------------------- | ------------------------------------- | ------------------------------------- |
| Where it runs       | Tier-0/Tier-1 edge                    | ESXi kernel, per-vNIC                 |
| What it filters     | North-south, inter-tier-1             | East-west, same-segment               |
| Use case            | Zone-to-zone, perimeter               | Microsegmentation, intra-zone         |
| Stateful inspection | Yes                                   | Yes                                   |
| L7 / IDS/IPS        | Yes (with Advanced Threat Prevention) | Yes (with Advanced Threat Prevention) |
| Performance impact  | Traffic must traverse edge            | Inline at vNIC, no hairpin            |

**Design principle**: Use the gateway firewall for coarse zone boundaries. Use the DFW for fine-grained workload isolation. They complement each other.

## Common Design Patterns

### Pattern 1: Environment Isolation (Most Common Starting Point)

Separate dev, staging, production at the tier-1 level. Apply DFW rules within each environment.

- **Effort**: Low
- **Value**: Prevents dev mistakes from impacting production
- **Gaps**: No intra-environment segmentation

### Pattern 2: Tiered Application Segmentation

Zone-based (web/app/db tiers) with DFW rules between tiers.

- **Effort**: Medium
- **Value**: Limits lateral movement within an environment
- **Gaps**: All web servers can still talk to all app servers (no app-level isolation)

### Pattern 3: Full Microsegmentation

Per-application rules using tag-based groups. Default deny everywhere.

- **Effort**: High (requires application dependency mapping)
- **Value**: Zero-trust east-west. Blast radius limited to a single application tier
- **Gaps**: Operational complexity. Requires good tagging discipline and dependency discovery

### Recommended Adoption Path

```
Start here                                              End state
    |                                                       |
    v                                                       v
Environment    →    Tiered within     →    Full micro-
isolation           each environment       segmentation
(tier-1 per env)    (DFW web/app/db)       (DFW per-app, default deny)
```

Don't jump to full microsegmentation on day one. Start with environment isolation, build operational muscle with DFW rules at the tier level, then progressively tighten to per-application policies as you map dependencies.

## Practical Considerations

### Dependency Discovery

You can't write firewall rules for traffic you don't understand. Before microsegmenting:

1. Enable DFW in **monitor-only mode** (log but don't block)
2. Use NSX Intelligence or vRealize Network Insight to map actual traffic flows
3. Build rules from observed traffic
4. Switch from monitor to enforce incrementally

### Rule Sprawl

The #1 operational risk. Mitigation:

- **Use groups, never raw IPs** in rules
- **One DFW section per application** -- owners manage their own rules
- **Automate rule creation** via NSX API or Terraform provider
- **Audit regularly** -- unused rules accumulate fast

### Performance

DFW adds minimal overhead (kernel-level, per-packet). In practice:

- Negligible latency impact for most workloads
- Rule table size matters more than rule count (keep rules specific, avoid "any/any" partial matches)
- Monitor `cpu used` on ESXi for DFW processing if running thousands of rules

### Migration from VLANs

If moving from VLAN-backed port groups to NSX segments:

1. Create NSX segments mirroring existing VLANs
2. Bridge NSX segments to VLANs (L2 bridge) during migration
3. Move VMs to NSX segments incrementally
4. Apply DFW rules in monitor mode first
5. Cut over routing from physical to tier-1
6. Remove VLAN bridges
