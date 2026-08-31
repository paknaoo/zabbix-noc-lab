# Troubleshooting: TLS Per-Host Encryption Setting Not Applied

**Phase:** 07 — Security Hardening

## Problem

After configuring certificates and TLS settings on both `zabbix-server` and `k8s-master` — including `TLSAccept=cert` on the agent and valid certificate paths in `zabbix_server.conf` — the agent rejected the server's connection attempts:

```
cannot accept unencrypted connection
```

This occurred despite both sides appearing correctly configured at the file level.

## Diagnosis

Zabbix's TLS configuration has two independent layers that must both be correct:

1. **File-level configuration** (`zabbix_server.conf`, `zabbix_agent2.conf`) — declares what the server and agent are *capable* of: which TLS mode, and where to find the relevant CA/certificate/key files.
2. **Per-host configuration in the Zabbix UI** (Data collection → Hosts → *host* → Encryption) — determines what the server *actually does* when connecting to that specific host: whether it uses plaintext or TLS for "Connections to host."

Having valid certificates and correct file-level settings on both ends is necessary but not sufficient — if the host's Encryption tab does not have "Certificate" selected and saved, Zabbix Server will still attempt a plaintext connection to that host, which the agent (correctly configured to require TLS) then rejects. In this case, the Encryption setting had apparently not been correctly saved on the first attempt.

## Resolution

1. Re-opened the host's Encryption tab in the Zabbix UI, re-selected "Certificate" for "Connections to host," and clicked **Update**.
2. Restarted `zabbix-server` to ensure the configuration syncer picked up the change without delay.
3. Verified directly with a TLS-flagged `zabbix_get`:
   ```bash
   zabbix_get -s 10.10.10.20 -p 10050 -k agent.ping --tls-connect cert \
     --tls-ca-file /etc/zabbix/pki/zabbix_ca.crt \
     --tls-cert-file /etc/zabbix/pki/zabbix_server.crt \
     --tls-key-file /etc/zabbix/pki/zabbix_server.key
   ```
   Returned `1`, confirming a working encrypted connection.

## Methodological Note

`zabbix_agent2.log` only records connection *errors*, not successful connections — after applying the fix, the absence of new log entries is itself a good sign (no new failures), not a cause for concern. Functional verification — Latest data in the UI, or a direct TLS-flagged `zabbix_get` — is more reliable evidence that a fix worked than log inspection alone.

## Portfolio Note

This demonstrates understanding of Zabbix's two-layer TLS model: file-level settings define capability, while the per-host UI setting defines actual behaviour for that specific host. The two must be kept in sync — a mismatch produces a failure that looks unexplained even though the file-level configuration is entirely correct, because the missing piece isn't in a config file at all.
