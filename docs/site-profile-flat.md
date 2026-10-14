# Site profile — flat layout

*How this template instantiates on a site whose IP plan is **flat**: one
management /24 carrying every BMC, switch, firewall, and PDU; one data
/24 per role (control / service / storage); wide /21s for the GPU node
planes. Pattern language — a site copy replaces the examples with its
real values in `variables.sh`.*

## The shape

- **Management /24 is shared by everything** and will run hot: PDUs
  alone can take 150+ addresses. Inventory the free pockets before
  assigning anything new (PBS host, monitoring) — there is no room to
  improvise later.
- **Gateways sit at `.254`** in every subnet. Follow the site convention;
  do not invent a second one.
- Node numbering **mirrors the physical index** in every plane (node *n*
  takes the same last-octet pattern on BMC, data, and any added VLAN).
  Keep it that way — it makes cross-plane lookups trivial at 2 AM.

## Proxmox tier sizing

- **3 nodes in the cluster** (the template default). Additional service
  hosts at the site are *not* cluster members — they run other tasks.
- **PBS goes on a dedicated 4th host, never as a VM on the cluster it
  protects.** A backup server inside its own failure domain is a
  bootstrap trap: cluster down → backups unreachable → restores dead on
  arrival. If the site must be convinced, that sentence is the argument.
  The PBS host does not join the cluster; quorum stays at 3.

## Ceph addressing (the flat variant)

The template's two dedicated Ceph /24s become **one existing network +
one added VLAN**:

| Role | Assignment |
|---|---|
| Ceph **public** | The **existing service/data VLAN** — use the nodes' existing bond IPs. No new addressing; hyperconverged clients are the hosts themselves. |
| Ceph **cluster** (replication) | **One new VLAN** on the same bond, taking the **next pair in the site's numbering pattern** (next role VLAN, next subnet). MTU 9000. **No gateway** — replication never routes. Node IPs mirror the physical index. |

Why the one deviation from flat earns its keep: replication and recovery
traffic is the bulkiest, most latency-sensitive flow in the design. A
tagged sub-interface costs nothing and keeps rebuild storms off the VM
traffic's fate.

**Fallback (zero new VLANs):** secondary cluster IPs on the service /24
at **public + 100** per node. Works, Ceph supports it; accept the
shared-fate caveat in writing.

**Corosync:** rides the service VLAN (flat sites typically have no
separate host-mgmt IPs). If the replication VLAN is added, register it
as corosync's second link.

## Netris bootstrap (if the site runs Netris)

- **Two switches stay permanently outside Netris management.** Netris
  controllers are VMs on the Proxmox hosts; if every path to the fabric
  runs through Netris-managed switches, a cold start has a circular
  dependency. The two unmanaged switches are the escape hatch.
- Controllers run **active-active across hosts** — do not pin the
  controller set to a single host (a stale "one host only" note in a
  site spreadsheet is a known artifact; the design intent is anywhere).
- Bring-up order: physical mgmt links → Proxmox → Netris VMs →
  Netris-managed fabric (also §02-network 3b).

## Phased bring-up

Hardware lead times mean **the management plane may be live weeks
before the data plane**. Sequence the build so mgmt-only operation
works for that stretch (BMC access, firmware baselines, Proxmox install
prep), and never let automation hard-depend on both planes existing at
once. Sections 00a/01 carry the ordering notes.

## Storage note

The site may have DDN gear racked and addressed — for this tier it is
**off-limits** (see Locked design decisions). Ceph on local NVMe is the
storage story; do not design dependencies on the DDN VLAN.

## GPU side (cross-reference)

InfiniBand touches the GPU nodes only. UFM servers are **physical**
(they need IB HCAs — never VMs) and are provisioned like GPU nodes
(BCM inventory + image, §16 of the BCM track); the BCM head node —
especially as a VM — has no fabric visibility. Fabric state is read
from UFM or the GPU nodes, never from the head node.
