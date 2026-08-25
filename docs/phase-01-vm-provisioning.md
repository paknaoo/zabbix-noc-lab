# Phase 01 — VM Provisioning and System Baseline

This phase covers provisioning the `zabbix-server` virtual machine and bringing it to a secure, up-to-date baseline before any Zabbix-specific software is installed.

---

## VM Provisioning

Ubuntu Server 24.04 LTS was installed via the Subiquity installer, with a static IP address configured directly during installation (see [Phase 00 — Planning](phase-00-planning.md) for the full specification). OpenSSH was installed during setup, without a key pre-loaded.

Basic connectivity was verified after first boot:

```bash
ping -c 4 192.168.50.254   # pfSense gateway
ping -c 4 1.1.1.1           # external IP reachability
ping -c 4 google.com         # DNS resolution
```

All three checks succeeded.

---

## SSH Key-Based Authentication

The existing SSH key already used across the lab (for `k8s-master`, `k8s-worker1`, `k8s-worker2` and `mgmt`) was reused for consistency:

```bash
ssh-copy-id adam@192.168.50.20
```

An alias was added to `~/.ssh/config` on `mgmt`:

```
Host zabbix
    HostName 192.168.50.20
    User adam
```

Password authentication was then disabled. This step surfaced a configuration precedence issue — see [Troubleshooting: SSH Password Authentication Overridden by cloud-init](troubleshooting/ssh-password-auth-cloud-init-override.md) for the full diagnosis and fix.

---

## System Updates

```bash
sudo apt update && sudo apt upgrade -y
```

A `needrestart` warning was raised (non-critical); resolved with a full reboot:

```bash
sudo reboot
```

---

## Host-Level Firewall (UFW)

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp
sudo ufw enable
```

Verified:

```bash
sudo ufw status verbose
```

Output confirmed `Status: active`, with `22/tcp ALLOW IN` as the only inbound rule.

---

## Time Synchronisation

`systemd-timesyncd` was already active and synchronised — no changes required:

```bash
timedatectl status
```

Confirmed `System clock synchronized: yes` and `NTP service: active`.

The timezone was changed from `Etc/UTC` to `Europe/London`:

```bash
sudo timedatectl set-timezone Europe/London
```

> **Note:** the rest of the `k8s-cilium-lab` hosts (`k8s-master`, both workers, `mgmt`) remain on `Etc/UTC`, despite the lab being physically located in the UK. This is a known inconsistency in that repository, outside the scope of `zabbix-noc-lab`. `zabbix-server` was configured correctly from the outset; aligning the rest of the lab is a potential future improvement tracked in `k8s-cilium-lab`, not here.

---

## Phase 01 Checkpoint

At the end of this phase:

- `zabbix-server` (`192.168.50.20`) is reachable exclusively via SSH key-based authentication.
- UFW is active, with SSH as the only permitted inbound service.
- The system is fully updated, rebooted, and time-synchronised with the correct local timezone.
- MySQL, Zabbix and Apache are not yet installed — this is covered in Phases 02–04.

See the [Validation](../README.md#validation) section of the README for the verification output captured for this phase.
