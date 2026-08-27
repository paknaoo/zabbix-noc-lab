# Troubleshooting: MySQL Binary Logging Blocks Zabbix Schema Import

**Phase:** 03 — Zabbix Server, Apache and PHP Installation

## Problem

Importing the Zabbix 7.0 database schema failed partway through:

```
ERROR 1419 (HY000) at line 2494: You do not have the SUPER privilege and binary logging is enabled
```

Re-running the same import command without first cleaning up the database then failed differently:

```
ERROR 1050 (42S01): Table 'role' already exists
```

## Diagnosis

Ubuntu 24.04's default MySQL installation has binary logging enabled. Creating certain objects in the Zabbix schema — functions and triggers — requires either the `SUPER` privilege or the `log_bin_trust_function_creators` setting to be enabled. Without either, MySQL refuses to create these objects while binary logging is active.

Because the import failed partway through, the database was left in a partial state: some tables (including `role`) had already been created before the failure. Re-running the same import against this partially-populated database inevitably collided with the tables that already existed.

## Resolution

1. Reset the database to a clean state:
   ```sql
   DROP DATABASE zabbix;
   CREATE DATABASE zabbix CHARACTER SET utf8mb4 COLLATE utf8mb4_bin;
   GRANT ALL PRIVILEGES ON zabbix.* TO 'zabbix'@'localhost';
   FLUSH PRIVILEGES;
   ```
2. Enabled the required setting **before** re-attempting the import:
   ```sql
   SET GLOBAL log_bin_trust_function_creators = 1;
   ```
3. Re-ran the import from scratch:
   ```bash
   zcat /usr/share/zabbix-sql-scripts/mysql/server.sql.gz | mysql -u zabbix -p zabbix
   ```
   Completed without errors.

## Portfolio Note

This demonstrates that a failed schema import cannot simply be resumed or re-run against a partially-populated database — it requires a clean restart of the database itself. It also provided practical, first-hand exposure to the interaction between MySQL's `SUPER` privilege model and binary logging, rather than encountering it only in documentation.

In hindsight, setting `log_bin_trust_function_creators` was considered as an optional "for good measure" step during [Phase 02 — MySQL Installation and Configuration](../phase-02-mysql.md), but was not applied at the time. This phase confirmed that, for this specific MySQL configuration, the setting is not optional — it is required for a successful Zabbix schema import.
