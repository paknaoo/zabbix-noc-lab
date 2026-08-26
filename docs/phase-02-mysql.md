# Phase 02 — MySQL Installation and Configuration

This phase covers installing and hardening MySQL as the Zabbix backend database, and creating a dedicated database and least-privilege user. It does not yet include the Zabbix schema import, which is deferred to Phase 03.

---

## Package Installation

```bash
sudo apt install mysql-server -y
```

Version installed:

```
$ mysql --version
mysql  Ver 8.0.46-0ubuntu0.24.04.3 for Linux on x86_64 ((Ubuntu))
```

Service status:

```
$ systemctl status mysql
● mysql.service - MySQL Community Server
     Loaded: loaded (/usr/lib/systemd/system/mysql.service; enabled; preset: enabled)
     Active: active (running) since Wed 2026-08-26 21:47:31 BST; 13min ago
    Process: 970 ExecStartPre=/usr/share/mysql/mysql-systemd-start pre
   Main PID: 995 (mysqld)
     Status: "Server is operational"
      Tasks: 38 (limit: 4543)
     Memory: 425.1M (peak: 436.0M)
        CPU: 7.227s
     CGroup: /system.slice/mysql.service
             └─995 /usr/sbin/mysqld
```

```
$ systemctl is-enabled mysql
enabled
```

---

## Hardening

Run via the standard MySQL hardening script:

```bash
sudo mysql_secure_installation
```

Choices made:

| Prompt | Choice |
|---|---|
| VALIDATE PASSWORD component | Skipped — a deliberate lab-scope decision, not intended for production use |
| Remove anonymous users | Yes |
| Disallow remote root login | Yes |
| Remove test database | Yes |
| Reload privilege tables | Yes |

---

## Dedicated Database and User

```sql
CREATE DATABASE zabbix CHARACTER SET utf8mb4 COLLATE utf8mb4_bin;
CREATE USER 'zabbix'@'localhost' IDENTIFIED BY '<redacted>';
GRANT ALL PRIVILEGES ON zabbix.* TO 'zabbix'@'localhost';
FLUSH PRIVILEGES;
```

**Design notes:**

- The user is scoped to `'zabbix'@'localhost'`, not `'%'`. The database is never accessed remotely — Zabbix Server connects to it locally on the same host — so restricting the user to `localhost` is a least-privilege choice rather than an oversight.
- `utf8mb4` / `utf8mb4_bin` is required by Zabbix 7.0's schema.
- The database password is not recorded in this repository. Verification below confirms the account works without exposing the credential.

---

## Verification

```
$ mysql -u zabbix -p zabbix -e "SELECT DATABASE() AS 'Database (zabbix)';"
+--------------------+
| Database (zabbix)  |
+--------------------+
| zabbix             |
+--------------------+
```

```
$ mysql -u root -p -e "SELECT User, Host FROM mysql.user WHERE User='zabbix';"
+--------+-----------+
| User   | Host      |
+--------+-----------+
| zabbix | localhost |
+--------+-----------+
```

```
$ mysql -u root -p -e "SHOW GRANTS FOR 'zabbix'@'localhost';"
+---------------------------------------------------------------+
| Grants for zabbix@localhost                                   |
+---------------------------------------------------------------+
| GRANT USAGE ON *.* TO `zabbix`@`localhost`                    |
| GRANT ALL PRIVILEGES ON `zabbix`.* TO `zabbix`@`localhost`    |
+---------------------------------------------------------------+
```

This confirms three things together: the `zabbix` user can authenticate and reach the `zabbix` database, the account exists solely with a `localhost` host scope (not `%`), and its privileges are limited to the `zabbix` schema rather than granted globally.

---

## Phase 02 Checkpoint

- MySQL installed and hardened; the service is `active` and `enabled`.
- The `zabbix` database exists with the required `utf8mb4` / `utf8mb4_bin` character set and collation.
- The `zabbix@localhost` user has full privileges scoped to the `zabbix` database only.
- The Zabbix SQL schema has **not** been imported yet — this is intentionally deferred to Phase 03, where the `zabbix-sql-scripts` package (installed alongside Zabbix Server) provides the schema file to import.

**Out of scope for this phase:**

- Importing the Zabbix SQL schema (Phase 03)
- Configuring `zabbix_server.conf` with database connection details (Phase 03)

See the [Validation](../README.md#validation) section of the README for a summary of the verification carried out for this phase.
