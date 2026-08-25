# Phase 00 — Planning

This phase documents the architectural decisions made before any infrastructure was provisioned, and the reasoning behind them. Unlike the implementation phases that follow, this document has no accompanying commands or validation output — its purpose is to record *why* the lab was designed the way it was.

---

## Network Placement

The Zabbix server is placed in the **OUTSIDE / management network** (`192.168.50.0/24`), rather than in the Kubernetes LAN (`10.10.10.0/24`) used by `k8s-cilium-lab`.

**Rationale:** this mirrors a realistic NOC (Network Operations Centre) pattern, in which the monitoring station lives in a separate management network from the systems it monitors. SNMP and Zabbix agent2 traffic to the monitored Kubernetes nodes must therefore cross a routed boundary and pass through firewall rules on pfSense. This deliberately introduces an additional exercise — configuring firewall rules at a network boundary — rather than monitoring "from inside" the same network as the Kubernetes cluster.

---

## Virtual Machine Specification

| Parameter | Value |
|---|---|
| Operating system | Ubuntu Server 24.04 LTS |
| Hostname | `zabbix-server` |
| IP address | `192.168.50.20/24` (static, set during installation) |
| Gateway | `192.168.50.254` (pfSense, OUTSIDE interface) |
| DNS | `1.1.1.1`, `8.8.8.8` |
| VMware network | VMnet11 (OUTSIDE), consistent with the `k8s-cilium-lab` convention |
| Username | `adam` |
| Resources | 2 vCPU / 4 GB RAM / 20 GB disk |
| Installation method | Bare metal — packages, systemd, MySQL, Apache and PHP. Deliberately not Docker, to demonstrate deeper systems administration knowledge. |

---

## Technology Stack

- **Zabbix 7.0 LTS**
- **Database:** MySQL
- **Frontend:** Apache and PHP, served over HTTPS with a self-signed certificate
- **SNMP v2c** — chosen over v3 as a deliberate simplicity trade-off at this stage of the lab. SNMPv3 (with authentication and encryption) was considered and is noted here as a possible future improvement.

---

## Monitored Infrastructure

All monitored hosts already exist in the [k8s-cilium-lab](https://github.com/paknaoo/k8s-cilium-lab) repository. `zabbix-noc-lab` does not modify that infrastructure — it only observes it.

| Host | IP address | Monitoring method |
|---|---|---|
| pfSense | `192.168.50.254` | SNMP v2c |
| k8s-master | `10.10.10.20` | Zabbix agent2 |
| k8s-worker1 | `10.10.10.21` | Zabbix agent2 |
| k8s-worker2 | `10.10.10.22` | Zabbix agent2 |
| mgmt | `192.168.50.10` | Zabbix agent2 |

Zabbix host naming follows the existing hostnames 1:1, with no additional prefixes.

---

## Firewall and Access Design

**New pfSense alias:** `HOST_ZABBIX` = `192.168.50.20`

**Rules implemented in Phase 00/01:**

| Source | Destination | Purpose |
|---|---|---|
| `HOST_ZABBIX` | Any | Outbound access (package installation, Zabbix repository) — modelled on the existing `HOST_MGMT → Any` rule documented in `k8s-cilium-lab`'s Phase 01 networking. |

**Rules planned for later phases (not yet implemented):**

| Source | Destination | Purpose | Phase |
|---|---|---|---|
| `HOST_ZABBIX` | `192.168.50.254:161/UDP` | SNMP to pfSense | 05 |
| `HOST_ZABBIX` | `10.10.10.0/24:10050/TCP` | agent2 to Kubernetes nodes | 06 |

**SNMP access on pfSense** will be restricted exclusively to `192.168.50.20/32`, rather than the whole OUTSIDE network — a deliberate least-privilege decision, consistent with the firewall philosophy used throughout `k8s-cilium-lab`.

**Zabbix frontend access** (Apache/PHP, Phase 04) will be restricted to `mgmt` (`192.168.50.10`) only, not the whole OUTSIDE network.

**Alerting** is out of scope for the initial build — only the dashboard and triggers will be configured. Alerting may be introduced as a later, separate phase.

---

## Host-Level Firewall (UFW)

UFW provides a second layer of filtering on `zabbix-server`, in addition to the pfSense rules — a defence-in-depth approach.

- Default policy: deny incoming, allow outgoing.
- The only inbound port currently open is `22/tcp` (SSH).
- Ports are opened incrementally, only once the relevant component (for example, Apache in Phase 04) actually requires inbound traffic — not pre-emptively.
