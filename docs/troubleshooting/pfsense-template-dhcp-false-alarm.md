# Troubleshooting: False DHCP Alarm After pfSense Template Swap

**Phase:** 06 — Triggers and NOC Dashboard

## Problem

After replacing the `pfsense` host's template from "Generic by SNMP" with the more specific "pfSense by SNMP" (to gain network throughput and firewall state table metrics — see [Phase 06](../phase-06-dashboards-triggers.md)), a new High-severity trigger fired immediately: **"PFSense: DHCP server is not running."**

## Diagnosis

The "pfSense by SNMP" template includes a trigger that assumes the DHCP server role is expected to be active on the device. This lab's infrastructure — both `k8s-cilium-lab` and `zabbix-noc-lab` — deliberately uses static IP addressing throughout. pfSense's DHCP server has never been running here, and is not intended to run; the trigger is not detecting a fault, it is flagging a mismatch between the template's default assumption and this architecture's deliberate design.

## Resolution

The trigger was **disabled** at the host level (Data collection → Hosts → `pfsense` → Triggers → "PFSense: DHCP server is not running" → Disabled), rather than being deleted, so the reasoning remains visible and the trigger can be trivially re-enabled if DHCP is ever introduced in the future.

## Portfolio Note

This is a small but useful example of tuning a pre-built template to a specific architecture rather than accepting every default trigger without review. A template written for the general case of "a pfSense device" will assume services that this particular, deliberately minimal lab does not use — recognising that mismatch, and disabling only the specific trigger affected (rather than the whole template, or ignoring the alert), keeps the monitoring signal meaningful.
