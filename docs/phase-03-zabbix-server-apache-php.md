# Phase 03 — Zabbix Server, Apache and PHP Installation

> **Note on phase numbering:** the original plan (see [Phase 00 — Planning](phase-00-planning.md)) treated the Zabbix Server installation and the frontend configuration as two separate phases. In practice, both were completed together in a single build session, since the frontend installation wizard depends directly on the Zabbix Server and database being already in place. This document reflects the actual build order; phase numbers from Phase 04 onward have been adjusted accordingly (see the [README](../README.md#documentation)).

This phase covers installing Zabbix Server 7.0 LTS, importing the database schema, configuring the Apache/PHP frontend over HTTPS, and restricting frontend access to `mgmt`.

---

## Adding the Zabbix 7.0 LTS Repository

```bash
wget https://repo.zabbix.com/zabbix/7.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_latest_7.0+ubuntu24.04_all.deb
sudo dpkg -i zabbix-release_latest_7.0+ubuntu24.04_all.deb
sudo apt update
```

## Package Installation

```bash
sudo apt install zabbix-server-mysql zabbix-frontend-php zabbix-apache-conf zabbix-sql-scripts zabbix-agent2 -y
```

`zabbix-agent2` was installed on `zabbix-server` itself, alongside the other packages — a deliberate decision so that the monitoring server also monitors itself (a standard NOC pattern: "who watches the watcher"). `zabbix-server` will be formally added as a monitored host in Phase 05, alongside the rest of the fleet.

---

## Database Schema Import

```bash
zcat /usr/share/zabbix-sql-scripts/mysql/server.sql.gz | mysql -u zabbix -p zabbix
```

The first import attempt failed partway through due to a MySQL binary logging permission issue. See [Troubleshooting: MySQL Binary Logging Blocks Zabbix Schema Import](troubleshooting/mysql-binlog-schema-import-failure.md) for the full diagnosis and fix. The import was re-run from a clean database and completed without errors.

---

## Zabbix Server Configuration

`/etc/zabbix/zabbix_server.conf`:

```
DBName=zabbix       # already present by default
DBUser=zabbix       # already present by default
DBPassword=<redacted>   # added manually
```

The database password is not recorded in this repository.

---

## Starting and Enabling Services

```bash
sudo systemctl restart zabbix-server zabbix-agent2 apache2
sudo systemctl enable zabbix-server zabbix-agent2 apache2
```

All three services confirmed `active (running)` and `enabled`.

---

## Zabbix Server Log Verification

`zabbix_server.log` confirmed a clean startup of all pollers and workers, with no `DBconnect` errors.

A number of `became not supported: No "X" processes started` messages appeared for the IPMI poller, Java poller, VMware collector, report writer/manager, and connector manager/worker. These are **expected** — they refer to optional Zabbix features that are deliberately not enabled in this installation, not errors.

One transient `first network error` was logged on the first connection attempt from Zabbix Server to its own local agent2 — a race condition caused by restarting both services at the same time. This resolved itself within roughly 16 seconds, confirmed by a subsequent `interface became available` log entry.

---

## PHP Configuration

`date.timezone` was **not** set automatically by the installation package, and had to be set manually in `/etc/php/8.3/apache2/php.ini`:

```bash
sudo sed -i 's|^;\?date.timezone =.*|date.timezone = Europe/London|' /etc/php/8.3/apache2/php.ini
```

Confirmed: `date.timezone = Europe/London`.

PHP is served through classic `mod_php`, not PHP-FPM — confirmed via `apache2ctl -M` (`php_module (shared)` present) and the absence of a `php8.3-fpm.service` unit on the system.

---

## Apache `ServerName` Warning Fix

```bash
echo "ServerName zabbix-server" | sudo tee /etc/apache2/conf-available/servername.conf
sudo a2enconf servername
sudo systemctl restart apache2
```

This removed the `Could not reliably determine the server's fully qualified domain name` startup warning.

---

## HTTPS with a Self-Signed Certificate

```bash
sudo mkdir -p /etc/ssl/zabbix
sudo openssl req -x509 -nodes -days 825 -newkey rsa:2048 \
  -keyout /etc/ssl/zabbix/zabbix-selfsigned.key \
  -out /etc/ssl/zabbix/zabbix-selfsigned.crt \
  -subj "/C=GB/ST=England/L=Sutton/O=zabbix-noc-lab/CN=zabbix-server"
sudo a2enmod ssl
```

A `:443` VirtualHost was created at `/etc/apache2/sites-available/zabbix-ssl.conf`:

```apache
<VirtualHost *:443>
    ServerName zabbix-server
    DocumentRoot /usr/share/zabbix

    SSLEngine on
    SSLCertificateFile /etc/ssl/zabbix/zabbix-selfsigned.crt
    SSLCertificateKeyFile /etc/ssl/zabbix/zabbix-selfsigned.key

    <Directory /usr/share/zabbix>
        Options FollowSymLinks
        AllowOverride None
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/zabbix-ssl-error.log
    CustomLog ${APACHE_LOG_DIR}/zabbix-ssl-access.log combined
</VirtualHost>
```

Enabled and verified:

```bash
sudo a2ensite zabbix-ssl.conf
sudo apache2ctl configtest   # Syntax OK
sudo systemctl reload apache2
```

---

## Zabbix Installation Wizard

Completed via the browser, from `mgmt`. The wizard passed all pre-requisite checks (including `date.timezone`, hence the earlier manual fix), connected successfully to the database (MySQL, `localhost`, socket connection, `zabbix`/`zabbix`), and configured the Zabbix server connection (`localhost:10051`). The wizard generated `/etc/zabbix/web/zabbix.conf.php` automatically.

The default credentials (`Admin` / `zabbix`) were used only to complete initial login, and the Admin password was **changed immediately** afterwards — the default password was not left active.

---

## Restricting Frontend Access to `mgmt`

```bash
sudo ufw delete allow 443/tcp
sudo ufw allow from 192.168.50.10 to any port 443 proto tcp comment 'Zabbix frontend HTTPS - mgmt only'
```

Confirmed via `ufw status verbose`: `443/tcp ALLOW IN 192.168.50.10` — not `Anywhere`.

**Architectural note:** this restriction was implemented at the **UFW (host-based firewall)** level, not on pfSense. `zabbix-server` and `mgmt` sit in the same OUTSIDE network segment (`192.168.50.0/24`), so traffic between them never crosses pfSense — there is no inter-subnet routing involved, and a pfSense rule would have no effect on this traffic. This is a useful illustration of the distinction between inter-network filtering (pfSense) and intra-network filtering (host firewall) within the same architecture — see [Architecture](architecture.md#access-filtering-pfsense-vs-ufw) for further detail.

A negative test — confirming that a host other than `mgmt` on the OUTSIDE network cannot reach port 443 — was **not performed** in this phase. At present, only two hosts exist on the OUTSIDE network (`mgmt` and `zabbix-server`), so there is no third host available from which to run the test. This is a deliberate, noted limitation of test coverage at this stage, not a gap in the configuration itself.

---

## Verification

```
$ systemctl status zabbix-server --no-pager
● zabbix-server.service - Zabbix Server
     Loaded: loaded (/usr/lib/systemd/system/zabbix-server.service; enabled; preset: enabled)
     Active: active (running) since Wed 2026-08-26 23:19:48 BST
   Main PID: 17609 (zabbix_server)
```

```
$ systemctl status apache2 --no-pager
● apache2.service - The Apache HTTP Server
     Loaded: loaded (/usr/lib/systemd/system/apache2.service; enabled; preset: enabled)
     Active: active (running) since Wed 2026-08-26 23:35:36 BST
```

```
$ systemctl status zabbix-agent2 --no-pager
● zabbix-agent2.service - Zabbix Agent 2
     Loaded: loaded (/usr/lib/systemd/system/zabbix-agent2.service; enabled; preset: enabled)
     Active: active (running) since Wed 2026-08-26 23:19:48 BST
...
Aug 26 23:19:47 zabbix-server zabbix_agent2[17615]: Validating configuration file "/etc/zabbix/zabbix_agent2.conf"
Aug 26 23:19:48 zabbix-server zabbix_agent2[17615]: Validation successful
Aug 26 23:19:48 zabbix-server zabbix_agent2[17624]: Starting Zabbix Agent 2 (7.0.30)
```

All three services confirmed `active (running)`, `enabled`, with clean startup logs.

```
$ sudo ufw status verbose
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), disabled (routed)
New profiles: skip

To                         Action      From
--                         ------      ----
22/tcp                     ALLOW IN    Anywhere                   # SSH
443/tcp                    ALLOW IN    192.168.50.10              # Zabbix frontend HTTPS - mgmt only
22/tcp (v6)                ALLOW IN    Anywhere (v6)              # SSH
```

Confirms `443/tcp` is restricted to `192.168.50.10` — not `Anywhere` — while SSH remains open as configured in Phase 01.

```
$ zabbix_server -V
zabbix_server (Zabbix) 7.0.30
```

```
$ curl -Ik https://192.168.50.20/ 
HTTP/1.1 200 OK
Date: Wed, 26 Aug 2026 23:09:19 GMT
Server: Apache/2.4.58 (Ubuntu)
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Cache-Control: no-store, no-cache, must-revalidate
Pragma: no-cache
X-Content-Type-Options: nosniff
X-XSS-Protection: 1; mode=block
X-Frame-Options: SAMEORIGIN
Set-Cookie: zbx_session=<redacted>; secure; HttpOnly
Content-Type: text/html; charset=UTF-8
```

Confirms the frontend serves `200 OK` over HTTPS from `mgmt`, with standard Zabbix security headers (`X-Content-Type-Options`, `X-Frame-Options`, `HttpOnly`/`secure` session cookie) present. The session cookie value itself is redacted here and was not recorded in the evidence folder.

```
$ php -v
PHP 8.3.6 (cli) (built: Jul 16 2026 18:30:41) (NTS)
Copyright (c) The PHP Group
Zend Engine v4.3.6, Copyright (c) Zend Technologies
    with Zend OPcache v8.3.6, Copyright (c), by Zend Technologies
```

```
$ grep -A1 "https://php.net/date.timezone" /etc/php/8.3/apache2/php.ini
; https://php.net/date.timezone
date.timezone = Europe/London
```

---

## Phase 03 Checkpoint

- Zabbix Server 7.0, Apache (`mod_php`) and PHP 8.3 all `active` and `enabled`.
- The `zabbix` database is fully imported and connected to Zabbix Server, confirmed via clean server logs.
- The frontend is available over HTTPS (self-signed certificate) at `https://192.168.50.20`, with access restricted to `mgmt` (`192.168.50.10`) via UFW.
- The default Admin password has been changed.
- `zabbix-agent2` is installed on `zabbix-server`, ready to be formally added as a monitored host in Phase 05.
- PHP timezone set to `Europe/London` (set manually; not configured automatically by the package).

See the [Validation](../README.md#validation) section of the README for a summary of the verification carried out for this phase.
