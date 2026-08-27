# Troubleshooting: pfSense Broad Rule Leaking Access to Self-Targeted Traffic

**Phase:** 04 — pfSense SNMP Monitoring

## Context

This project's methodology requires every access restriction to be confirmed with a negative test, not just a positive one: it is not enough to prove that the intended host *can* reach a service — it must also be proven that hosts outside the intended scope *cannot*. The SNMP access rule added for `zabbix-server` in this phase (`HOST_ZABBIX → This Firewall : 161/UDP`) was believed to correctly restrict SNMP access to `zabbix-server` alone.

Following that methodology, a negative test was run from `mgmt` (`192.168.50.10`), which has no SNMP-specific rule and was therefore expected to be rejected:

```bash
snmpwalk -v2c -c zbx-noc-r0 192.168.50.254 system
```

## Problem

The negative test failed to fail: `mgmt` successfully retrieved full SNMP data from pfSense, despite there being no rule explicitly permitting SNMP access for `mgmt`.

## Diagnosis

pfSense evaluates firewall rules on an interface from top to bottom, and the **first matching rule wins** — later rules are never evaluated for that packet, regardless of how specific they are.

The OUTSIDE interface already contained a rule inherited from `k8s-cilium-lab`:

```
Source: HOST_MGMT | Protocol: * | Destination: * | Port: *
```

This rule was originally intended to mean "allow `mgmt` outbound access to the internet." In pfSense, however, `Destination: *` matches literally everything reachable from that interface — **including "This Firewall (self)"**. Because this broad rule was positioned above the newly-added SNMP restriction, any traffic from `mgmt` — including SNMP requests to pfSense's own SNMP daemon on port 161 — matched the broad rule and was passed, before the packet ever reached the rule intended to restrict SNMP to `HOST_ZABBIX`.

**This was not a fault in the new SNMP rule.** The SNMP rule for `HOST_ZABBIX` was correctly scoped. The issue was that a pre-existing, broad rule from a different repository (`k8s-cilium-lab`) had a wider practical scope than its original intent suggested, and rule ordering caused it to leak access to a service it was never meant to reach.

## Options Considered

**Option A — Narrow the existing broad rule.** Restrict `HOST_MGMT → *` by excluding "This Firewall" as a destination (e.g. `Destination: !This Firewall`), fixing the issue at its source. This was **not** chosen: it would mean modifying an existing, working rule that belongs to the `k8s-cilium-lab` repository's architecture, which is outside the scope of `zabbix-noc-lab`.

**Option B — Add an explicit, narrowly-scoped Block rule** above the existing broad rule, blocking only the specific traffic that should not be allowed. This was the option chosen.

## Resolution

An explicit Block rule was added above the pre-existing `HOST_MGMT → Any` rule:

```
Action: Block | Protocol: UDP | Source: HOST_MGMT | Destination: This Firewall (self) | Port: 161 (SNMP)
Description: "Block mgmt access to pfSense SNMP - explicit exclusion, see zabbix-noc-lab phase-04"
```

This keeps the fix entirely within the scope of `zabbix-noc-lab`, leaves the `k8s-cilium-lab` architecture untouched, and documents the exclusion explicitly via the rule description, rather than relying on rule-order alone to communicate intent to a future reader.

Final rule order on the OUTSIDE interface (relevant portion):

| # | Source | Protocol | Destination | Port | Action | Description |
|---|---|---|---|---|---|---|
| 1–5 | *(unchanged — existing `HOST_MGMT` rules from `k8s-cilium-lab`)* | | | | | WireGuard, WebGUI, ICMP, SSH to Kubernetes nodes, Kubernetes API |
| 6 | `HOST_MGMT` | UDP | This Firewall | 161 (SNMP) | **Block** | Block mgmt access to pfSense SNMP — explicit exclusion, see `zabbix-noc-lab` phase-04 |
| 7 | `HOST_MGMT` | * | * | * | Pass | Allow HOST_MGMT outbound *(pre-existing, `k8s-cilium-lab`)* |
| 8 | `HOST_ZABBIX` | * | * | * | Pass | Allow HOST_ZABBIX outbound |
| 9 | `HOST_ZABBIX` | UDP | This Firewall | 161 (SNMP) | Pass | Allow SNMP from zabbix-server to pfSense |

## Verification

The negative test was repeated after the fix, from `mgmt`:

```
$ snmpwalk -v2c -c zbx-noc-r0 192.168.50.254 .1.3.6.1.2.1.1
Timeout: No Response from 192.168.50.254
```

Confirmed: `mgmt` no longer has SNMP access to pfSense, while `zabbix-server` continues to poll successfully (see [Phase 04 — pfSense SNMP Monitoring](../phase-04-pfsense-snmp.md) for the successful `snmpwalk` from `zabbix-server`).

## Portfolio Note

This is the most significant finding in the project so far, for three reasons:

1. **Methodological discipline.** The issue was found purely because a negative test was run as a matter of routine, not because of a suspicion or a symptom. A configuration that "worked" from a positive-test perspective was in fact insecure.
2. **Understanding of firewall rule mechanics.** The root cause required recognising that rule evaluation is order-dependent and first-match-wins, and that a `Destination: *` rule includes self-targeted traffic to the firewall itself — a detail that is easy to overlook when a rule's description states a narrower intent than its actual match scope.
3. **Deliberate, documented trade-off between options.** Rather than defaulting to "fix it wherever is easiest," the less invasive option was chosen specifically to respect the boundary between this repository and `k8s-cilium-lab`, with the reasoning made explicit rather than left implicit in the rule order alone.
