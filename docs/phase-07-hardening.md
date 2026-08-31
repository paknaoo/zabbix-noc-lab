# Phase 07 — Security Hardening

This is the final phase of the project's originally planned scope (see [Phase 00 — Planning](phase-00-planning.md)). It covers three areas: forcing HTTP→HTTPS redirection on the frontend, adding Apache security headers, and encrypting all Zabbix agent2 traffic with TLS certificates issued from a lab-internal Certificate Authority.

---

## HTTP → HTTPS Redirect

Configured on the port-80 VirtualHost (`/etc/apache2/sites-available/000-default.conf`):

```apache
<VirtualHost *:80>
    ServerName zabbix-server
    RewriteEngine On
    RewriteCond %{HTTPS} off
    RewriteRule ^(.*)$ https://%{HTTP_HOST}%{REQUEST_URI} [L,R=301]
</VirtualHost>
```

> **Evidence note:** the terminal session used to capture this file's exact on-disk contents was interrupted before completing (the host restarted mid-capture). The configuration above is reconstructed from the build session's own record of what was applied, not from a captured file dump — flagged here for transparency rather than presented as a verified file capture, consistent with this repository's evidence standards.

`%{HTTP_HOST}` was used deliberately, rather than a hardcoded hostname, so the redirect works correctly whether a client requests the frontend by hostname or by IP address — avoiding a potential mismatch with the self-signed certificate's `CN=zabbix-server`.

Required module: `sudo a2enmod rewrite`.

**Verification:**

```
$ curl -Ik http://192.168.50.20
HTTP/1.1 301 Moved Permanently
Location: https://192.168.50.20/
```

---

## Apache Security Headers

Required module: `sudo a2enmod headers`.

Added inside the `<VirtualHost *:443>` block in `/etc/apache2/sites-available/zabbix-ssl.conf`:

```apache
Header always set Strict-Transport-Security "max-age=63072000; includeSubDomains"
Header always set X-Frame-Options "SAMEORIGIN"
Header always set X-Content-Type-Options "nosniff"
Header always set Referrer-Policy "strict-origin-when-cross-origin"
Header always set X-XSS-Protection "1; mode=block"
```

**Verification:**

```
$ curl -Ik https://192.168.50.20 --insecure
HTTP/1.1 200 OK
Date: Sun, 30 Aug 2026 23:40:08 GMT
Server: Apache/2.4.58 (Ubuntu)
Strict-Transport-Security: max-age=63072000; includeSubDomains
X-Frame-Options: SAMEORIGIN
X-Content-Type-Options: nosniff
Referrer-Policy: strict-origin-when-cross-origin
X-XSS-Protection: 1; mode=block
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Cache-Control: no-store, no-cache, must-revalidate
Pragma: no-cache
X-Content-Type-Options: nosniff
X-XSS-Protection: 1; mode=block
X-Frame-Options: SAMEORIGIN
Set-Cookie: zbx_session=<redacted>; secure; HttpOnly
Content-Type: text/html; charset=UTF-8
```

All five headers are present. **Note the duplication** of `X-Content-Type-Options`, `X-XSS-Protection` and `X-Frame-Options` in the response — the Zabbix frontend already sets its own copies of these headers at the PHP application level, independent of the Apache configuration added here. This is harmless (browsers use the consistent values from both sources) and is a useful illustration of layered defence: the Apache-level configuration adds server-level protection regardless of what the application does. Separately, the `zbx_session` cookie is confirmed to carry `secure` and `HttpOnly` natively — a property of Zabbix itself, not of this configuration, but worth recording as a positive existing characteristic of the frontend.

---

## TLS with a Self-Managed PKI for Agent2 Communication

**Design decision: certificates over PSK.** Zabbix recommends PSK as the lighter-weight option for agent-server encryption. Certificates (a full PKI) were chosen instead, despite the greater setup effort across five hosts, because PKI better demonstrates certificate management skills — running a CA, signing CSRs, distributing certificates — and more closely resembles real enterprise deployments than a shared-secret approach.

**Rollout approach:** implemented and verified end-to-end on a single host (`k8s-master`) before repeating the process on the rest (`k8s-worker1` and `k8s-worker2` together, then `mgmt` separately) — a deliberate choice to minimise risk when making the same change across multiple hosts.

### Creating the Certificate Authority

On `zabbix-server`:

```bash
mkdir -p /etc/zabbix/pki
openssl genrsa -out zabbix_ca.key 4096
openssl req -x509 -new -nodes -key zabbix_ca.key -sha256 -days 3650 \
  -out zabbix_ca.crt -subj "/C=GB/ST=England/L=Sutton/O=zabbix-noc-lab/CN=zabbix-noc-lab-CA"
```

### Server Certificate

Signed by the new CA. Configured in `/etc/zabbix/zabbix_server.conf`:

```
TLSCAFile=/etc/zabbix/pki/zabbix_ca.crt
TLSCertFile=/etc/zabbix/pki/zabbix_server.crt
TLSKeyFile=/etc/zabbix/pki/zabbix_server.key
```

### Per-Agent Certificates

The same pattern was applied to all four remaining hosts (`k8s-master`, `k8s-worker1`, `k8s-worker2`, `mgmt`):

1. Generate a key, a CSR, and sign it with the CA — on `zabbix-server`.
2. Distribute via `mgmt` as an intermediary: public `.crt` files copied directly by SCP; private `.key` files required a workaround via a temporary copy to `/home/adam` with a `chown`, due to their `600` permissions and `zabbix:zabbix` ownership.
3. Place on the target host under `/etc/zabbix/pki/`, with correct permissions (`600` for `.key`, `644` for `.crt`, owned by `zabbix:zabbix`).
4. Configure `TLSConnect=cert`, `TLSAccept=cert` and the CA/cert/key paths in `zabbix_agent2.conf`. Example, `k8s-master`:
   ```
   TLSConnect=cert
   TLSAccept=cert
   TLSCAFile=/etc/zabbix/pki/zabbix_ca.crt
   TLSCertFile=/etc/zabbix/pki/k8s-master.crt
   TLSKeyFile=/etc/zabbix/pki/k8s-master.key
   ```
5. Restart `zabbix-agent2`.
6. In the Zabbix UI (**Data collection → Hosts → *host* → Encryption**), set both "Connections to host" (passive mode, the direction actually used) and "Connections from host" (active mode, unused in this architecture but closed off pre-emptively) to **Certificate**.

For `k8s-worker1` and `k8s-worker2`, a small bash loop generated both hosts' certificates in a single pass — a simple example of automating a repetitive task rather than repeating the same commands by hand four times.

**Final PKI directory on `zabbix-server`:**

```
$ ls -la /etc/zabbix/pki/
total 44
drwxr-xr-x 2 root   root   4096 Aug 31 00:35 .
drwxr-xr-x 6 root   root   4096 Aug 30 22:37 ..
-rw-r--r-- 1 root   root   1549 Aug 30 22:41 k8s-master.crt
-rw-r--r-- 1 root   root   1549 Aug 30 23:40 k8s-worker1.crt
-rw-r--r-- 1 root   root   1549 Aug 30 23:40 k8s-worker2.crt
-rw-r--r-- 1 root   root   1541 Aug 31 00:16 mgmt.crt
-rw-r--r-- 1 root   root   2025 Aug 30 22:28 zabbix_ca.crt
-rw------- 1 zabbix zabbix 3272 Aug 30 22:27 zabbix_ca.key
-rw-r--r-- 1 root   root     41 Aug 31 00:16 zabbix_ca.srl
-rw-r--r-- 1 root   root   1554 Aug 30 22:32 zabbix_server.crt
-rw------- 1 zabbix zabbix 1704 Aug 30 22:31 zabbix_server.key
```

Note the absence of any agent `.key` files here — see [Private Key Cleanup](#private-key-cleanup) below.

---

## Incident: Per-Host Encryption Setting Not Propagated

After the first full configuration of `k8s-master` (certificates, `zabbix_agent2.conf`, and the UI Encryption setting), the agent rejected the server's connection attempts with `cannot accept unencrypted connection`, despite `TLSAccept=cert` being correctly set on the agent and `zabbix_server.conf` correctly pointing to valid certificate paths.

Having both the server and agent *capable* of TLS is not sufficient — Zabbix Server separately decides, per host, *whether to actually use* TLS when connecting, based on the host's Encryption tab in the UI. That setting had apparently not been correctly saved on the first attempt.

See [Troubleshooting: TLS Per-Host Encryption Setting Not Applied](troubleshooting/tls-per-host-encryption-setting-not-applied.md) for the full diagnosis and fix.

---

## Negative Test: TLS Is Enforced, Not Merely Permitted

```
$ zabbix_get -s 10.10.10.20 -p 10050 -k agent.ping
Get value error: cannot read from socket: [104] Connection reset by peer
```

Run without TLS flags, this confirms `TLSAccept=cert` on the agents actively **rejects** unencrypted connections, rather than merely accepting both encrypted and unencrypted traffic.

---

## Private Key Cleanup

Once full functionality was confirmed across all four hosts, the local copies of the agents' private keys (`k8s-master.key`, `k8s-worker1.key`, `k8s-worker2.key`, `mgmt.key`) were removed from `/etc/zabbix/pki/` on `zabbix-server` — consistent with standard PKI practice: a CA signs certificates for the entities it issues them to, but does not retain permanent copies of those entities' private keys. Retained on `zabbix-server`: `zabbix_ca.key`/`.crt` (the CA's own key/certificate), `zabbix_server.key`/`.crt` (legitimately needed locally), and the public `.crt` files of all four agents (harmless to keep, for reference). Temporary copies used during distribution were also confirmed removed from `/tmp` on `mgmt`.

---

## Final Verification (All Four Agent2 Hosts)

```
$ for ip in 10.10.10.20 10.10.10.21 10.10.10.22 192.168.50.10; do
    echo "=== $ip ==="
    zabbix_get -s $ip -p 10050 -k agent.ping --tls-connect cert \
      --tls-ca-file /etc/zabbix/pki/zabbix_ca.crt \
      --tls-cert-file /etc/zabbix/pki/zabbix_server.crt \
      --tls-key-file /etc/zabbix/pki/zabbix_server.key
  done
=== 10.10.10.20 ===
1
=== 10.10.10.21 ===
1
=== 10.10.10.22 ===
1
=== 192.168.50.10 ===
1
```

All four monitored agent2 hosts now communicate with Zabbix Server exclusively over an encrypted connection, authenticated by a certificate from the lab's own internal CA.

---

## Phase 07 Checkpoint

- Frontend: HTTP forces a redirect to HTTPS (301); five security headers active and confirmed present.
- Agent2 communication (all four hosts) fully encrypted with certificate authentication; confirmed by a negative test showing unencrypted connections are actively rejected.
- A self-managed CA (`zabbix-noc-lab-CA`), valid for 10 years, with its private key existing only on `zabbix-server`.
- `/etc/zabbix/pki/` on `zabbix-server` cleaned up — no agent private keys remain after distribution.

This closes the full, originally planned scope of the project (Phases 00–07). The project is now functionally complete.

See the [Validation](../README.md#validation) section of the README for a summary of the verification carried out for this phase.
