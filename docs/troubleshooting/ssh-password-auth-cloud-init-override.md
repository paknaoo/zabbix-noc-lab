# Troubleshooting: SSH Password Authentication Overridden by cloud-init

**Phase:** 01 — VM Provisioning and System Baseline

## Problem

After setting `PasswordAuthentication no` in `/etc/ssh/sshd_config` and restarting the SSH service, password-based login was still possible.

## Diagnosis

Ubuntu 24.04, via cloud-init, creates `/etc/ssh/sshd_config.d/50-cloud-init.conf` containing `PasswordAuthentication yes`. The main `sshd_config` file ends with:

```
Include /etc/ssh/sshd_config.d/*.conf
```

Directives loaded via `Include` are read *after* the main configuration file, and therefore override earlier settings — including the `PasswordAuthentication no` set directly in `sshd_config`.

## Resolution

`PasswordAuthentication` was changed to `no` directly inside `/etc/ssh/sshd_config.d/50-cloud-init.conf`, rather than deleting the file (which cloud-init may recreate on a future boot):

```bash
sudo nano /etc/ssh/sshd_config.d/50-cloud-init.conf
# PasswordAuthentication no
```

The configuration was validated before restarting the service:

```bash
sudo sshd -t
sudo systemctl restart ssh
```

## Confirmation

Password authentication was confirmed disabled by forcing a key-only connection attempt from both the hostname alias and the raw IP address:

```bash
ssh -o PubkeyAuthentication=no zabbix
ssh -o PubkeyAuthentication=no 192.168.50.20
```

Both attempts returned `Permission denied (publickey)`, confirming that password authentication was no longer accepted.

## Portfolio Note

This is a useful example of understanding configuration file load order and cloud-init drop-in behaviour, rather than simply following a tutorial command. A single `PasswordAuthentication no` line in the "obvious" location is not sufficient on a cloud-init-provisioned Ubuntu 24.04 host.
