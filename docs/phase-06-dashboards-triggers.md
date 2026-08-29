# Phase 06 — Triggers and NOC Dashboard

This phase covers building a single-pane-of-glass NOC dashboard, tuning the trigger set inherited from the assigned Zabbix templates, and deliberately deciding what *not* to add. Per the [Phase 00 plan](phase-00-planning.md), alerting (email/webhook notifications) remains explicitly out of scope here — this phase covers only triggers and the visual dashboard.

The work was carried out in three parts within a single build session; they are documented here as sub-sections of one phase rather than as separate phases, since they form one continuous, incremental piece of work.

---

## Part 1 — Dashboard Foundation

A new dashboard, **`NOC Overview`**, was created (Dashboards → Create dashboard) with two initial widgets:

**Problems widget:**
- Show: Recent problems
- Sort: severity
- Host groups: all (no filter applied)

**Host availability widget:**
- Host groups: all
- Displays an aggregate availability table broken down by interface type (Agent active/passive, SNMP, JMX, IPMI)

**Verification after Part 1:** all 6 hosts showed `Available`, with zero `Not available`, `Mixed` or `Unknown`. Two informational Warning problems ("Linux: Operating system description has changed" on `k8s-master` and `k8s-worker2`) were immediately visible — addressed in Part 3.

---

## Part 2 — Performance Visualisations

**Discovery during preparation:** the "Generic by SNMP" template assigned to `pfsense` in Phase 04 does **not** include any Low-Level Discovery (LLD) rule for network interfaces — confirmed by an empty Discovery rules list (`No data found`) and a review of the host's full 12-item list, none of which related to throughput. That template covers only ICMP checks, system information (description/location/name/contact/object ID), uptime, and SNMP availability.

**Decision:** "Generic by SNMP" was **replaced** with **"pfSense by SNMP"** — a platform-specific template — rather than adding it alongside the existing one. Assigning both simultaneously caused a conflict (overlapping definitions for the same OIDs/items), resolved by swapping templates rather than duplicating them.

**Benefit of the swap — new items, considerably richer than expected:**

- `Interface OUTSIDE(): Bits received/sent`, `Interface WAN(): Bits received/sent` — throughput, the original goal
- `States table current`, `States table limit`, `States table utilization in %` — packet filter metrics, an unplanned bonus valuable for a NOC context (an early signal of load or a potential scan/attack)

**Graph widgets created:**

1. **`k8s Nodes - CPU Utilization`** — Graph, item "CPU utilization" for `k8s-master`, `k8s-worker1`, `k8s-worker2` on a single chart
2. **`k8s Nodes - Memory Utilization`** — Graph, item "Available memory in %" for the same three hosts
3. **`pfSense - Network Traffic (OUTSIDE)`** — Graph, `Interface [em1(OUTSIDE)]: Bits received/sent`
4. **`pfSense - Firewall State Table Utilization`** — Graph, `States table utilization in %`

### Problem Found and Fixed: False DHCP Alarm

Swapping the template introduced a new trigger: **"PFSense: DHCP server is not running"** (High severity), immediately active as soon as the template was applied.

This project's entire infrastructure (`k8s-cilium-lab` and `zabbix-noc-lab`) deliberately uses static IP addressing throughout — pfSense's DHCP server has never been, and is not intended to be, running. The generic template's trigger assumes DHCP *should* be running, which doesn't match this lab's architecture.

See [Troubleshooting: False DHCP Alarm After pfSense Template Swap](troubleshooting/pfsense-template-dhcp-false-alarm.md) for the fix and reasoning.

---

## Part 3 — Refinement

**The remaining two Warning problems** (`k8s-master`, `k8s-worker2` — "Linux: Operating system description has changed") were identified as an expected side effect of the `apt upgrade` runs carried out in Phases 01 and 05 (a change in the reported kernel/distribution description following package updates). These were **acknowledged**, with a comment explaining the cause, rather than having their trigger disabled — a deliberate choice, since a future *unexpected* OS description change could be a meaningful security signal (for example, an unauthorised system change) worth continuing to watch for.

**Decision on custom triggers:** adding triggers beyond those provided by the official assigned templates ("pfSense by SNMP", "Linux by Zabbix agent") was **deliberately not pursued**. The existing set (54–126 triggers per host, depending on the number of dynamically discovered resources) already covers the key NOC scenarios: availability, performance, configuration drift, and host restarts. Adding custom triggers without a specific, justified need would have been scope creep for its own sake — choosing to rely on a well-considered, pre-built set is as much a deliberate decision as choosing to extend it would have been, when a real gap exists.

**Final dashboard layout** (top to bottom, general-to-specific):

1. Problems
2. Host availability
3. k8s Nodes — CPU Utilization | Memory Utilization (side by side)
4. pfSense — Network Traffic | Firewall State Table Utilization (side by side)

![NOC Overview dashboard](assets/phase-06/noc-overview-dashboard.png)

---

## Phase 06 Checkpoint

- Dashboard `NOC Overview` fully functional, 6 widgets, live data flowing.
- `pfsense` host template: **"pfSense by SNMP"** (replacing "Generic by SNMP"), with one trigger deliberately disabled (DHCP — see linked troubleshooting doc).
- Both pre-existing Warning problems (OS description changed) addressed via Acknowledge, with an explanatory comment.
- No active, unaddressed problems on the dashboard at the close of this phase.
- A deliberate decision was made not to add custom triggers, relying instead on a considered selection of official templates.

Alerting (notifications) remains out of scope, per the original Phase 00 plan — see the [Validation](../README.md#validation) section of the README for a summary of the verification carried out for this phase.
