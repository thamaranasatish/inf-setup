### FILE: execution-plan/12-phase-8b-vm10-sqlserver-install.md

# Phase 8b — VM10 Microsoft SQL Server 2022 Install

## Objective
Install Microsoft SQL Server 2022 on VM10 per Microsoft official guidance. OS selection (RHEL or Windows Server) must be confirmed by the client.

## Steps (Linux / RHEL path)

1. Install SQL Server 2022 on RHEL using the official Linux setup guide.
   - Official doc: https://learn.microsoft.com/en-us/sql/linux/sql-server-linux-setup
   - Official doc: https://learn.microsoft.com/en-us/sql/linux/quickstart-install-connect-red-hat

2. Install the SQL Server command-line tools (`sqlcmd`, `bcp`).
   - Official doc: https://learn.microsoft.com/en-us/sql/linux/sql-server-linux-setup-tools

3. Configure the edition, SA password, data directories, tempdb, backup directory, and memory per the client-approved design.
   - Official doc: https://learn.microsoft.com/en-us/sql/linux/sql-server-linux-configure-mssql-conf

4. Configure firewall, listener port (default TCP 1433), and service account.
   - Official doc: https://learn.microsoft.com/en-us/sql/linux/sql-server-linux-setup

## Steps (Windows Server path)

1. Install SQL Server 2022 on Windows Server per the official installation guide.
   - Official doc: https://learn.microsoft.com/en-us/sql/database-engine/install-windows/install-sql-server

2. Configure SQL Server services, collation, authentication, and security per official guidance.
   - Official doc: https://learn.microsoft.com/en-us/sql/database-engine/install-windows/

## Validation / Expected Results

- `sqlcmd -S <sql-host> -Q "SELECT @@VERSION"` returns SQL Server 2022 version string.
  - Official doc: https://learn.microsoft.com/en-us/sql/tools/sqlcmd/sqlcmd-utility
- SQL Server service is `active (running)` / `Started` per chosen OS.
- Backups configured per client policy.
  - Official doc: https://learn.microsoft.com/en-us/sql/relational-databases/backup-restore/backup-and-restore-of-sql-server-databases

## Exit Criteria

- SQL Server 2022 installed, reachable, and integrated with Jenkins (via Redgate toolchain where required).
