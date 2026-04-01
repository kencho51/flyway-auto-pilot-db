# Flyway CLI Usage

Flyway CLI is installed at `D:\flyway-12.2.0\`.

---

## Licensing

### Edition requirements

| Feature | Edition needed |
|---|---|
| `info`, `migrate`, `validate`, `clean`, `repair`, `baseline` | Community (free, no auth) |
| `undo` | **Teams** |
| `snapshot` | **Enterprise** |

For the standard migration workflow, Community Edition is fully sufficient.

### Online auth (standard)

```powershell
flyway auth -IAgreeToTheEula
```

Requires outbound access to `auth.red-gate.com` and `permits.red-gate.com`. On success the token is cached in `%APPDATA%\Redgate\` and auth persists across sessions.

> If your machine shows `It looks like you are offline` — Redgate's auth servers are blocked by your firewall. Ask your network team to whitelist `auth.red-gate.com` and `permits.red-gate.com`.

### Offline permit (air-gapped machines)

If the network cannot be unblocked, offline permits (`D:\flyway-cli-permit-31May26.dat`) are issued from https://permits.red-gate.com/offline?productCode=63, but **Flyway CLI 12.x has no CLI flag or config key to apply them**. The `auth` command is online-only in this version.

Alternative for Teams/Enterprise features on offline machines:
- Use **Flyway Desktop** (Antigravity) instead — it is already authenticated via its own `permit.jwt` and supports `undo` through the UI.

The permit file `flyway-cli-permit-31May26.dat` expires **31 May 2026**.

---

## Setup: add Flyway CLI to PATH (once)

Run this in PowerShell to add it for the current session:

```powershell
❯ $env:PATH = "D:\flyway-12.2.0;$env:PATH"
❯   
```

To make it permanent (user-level):

```powershell
[Environment]::SetEnvironmentVariable("PATH", "D:\flyway-12.2.0;" + [Environment]::GetEnvironmentVariable("PATH","User"), "User")
```

After that, restart your terminal and `flyway info` will work without the full path.

---

## Default environment

The default environment is set to **`development`** in your local `flyway.user.toml` (gitignored):

```toml
[flyway]
environment = "development"
```

Running any command without `-environment` will target `AutopilotDev`.

---

## Environments

| Flag | Database | Notes |
|---|---|---|
| *(none)* | AutopilotDev | Default — your local dev DB |
| `-environment=shadow` | AutopilotShadow | Managed by Flyway Desktop; avoid running manually |
| `-environment=Build` | AutopilotBuild | CI — clean-provisioned on every run |
| `-environment=Check` | AutopilotCheck | Reporting / QA — clean-provisioned |
| `-environment=Test` | AutopilotTest | Pre-production |
| `-environment=Prod` | AutopilotProd | Production |

---

## Common commands

All commands must be run from the project root (`D:\Flyway Autopilot`) so that `flyway.toml` is picked up automatically.

### `info` — show migration status

```powershell
# Default (development)
 ❯ flyway info                                                         14:09:46
Flyway Community Edition 12.2.0 by Redgate

See release notes here: https://help.red-gate.com/help/flyway-cli12/help_2.aspx?topic=release-notes-and-older-versions/release-notes-for-flyway-engine
Database: jdbc:sqlserver://10.194.136.125:40000;connectRetryInterval=10;connectRetryCount=1;maxResultBuffer=-1;sendTemporalDataTypesAsStringForBulkCopy=true;delayLoadingLobs=true;useFmtOnly=false;cacheBulkCopyMetadata=false;bulkCopyForBatchInsertAllowEncryptedValueModifications=false;bulkCopyForBatchInsertTableLock=false;bulkCopyForBatchInsertKeepNulls=false;bulkCopyForBatchInsertKeepIdentity=false;bulkCopyForBatchInsertFireTriggers=false;bulkCopyForBatchInsertCheckConstraints=false;bulkCopyForBatchInsertBatchSize=0;useBulkCopyForBatchInsert=false;cancelQueryTimeout=-1;sslProtocol=TLS;calcBigDecimalPrecision=false;useDefaultJaasConfig=false;jaasConfigurationName=SQLJDBCDriver;statementPoolingCacheSize=0;serverPreparedStatementDiscardThreshold=10;enablePrepareOnFirstPreparedStatementCall=false;fips=false;socketTimeout=0;authentication=sqlPassword;authenticationScheme=nativeAuthentication;xopenStates=false;datetimeParameterType=datetime2;sendTimeAsDatetime=true;replication=false;trustStoreType=JKS;trustServerCertificate=true;TransparentNetworkIPResolution=true;iPAddressPreference=IPv4First;serverNameAsACE=false;sendStringParametersAsUnicode=true;selectMethod=direct;responseBuffering=adaptive;queryTimeout=-1;packetSize=8000;multiSubnetFailover=false;loginTimeout=30;lockTimeout=-1;lastUpdateCount=true;useDefaultGSSCredential=false;prepareMethod=prepexec;encrypt=mandatory;disableStatementPooling=true;databaseName=AutopilotDev;columnEncryptionSetting=Disabled;applicationName=Microsoft JDBC Driver for SQL Server;applicationIntent=readwrite; (Microsoft SQL Server 15.0)
ERROR: Skipping filesystem location: callbacks (not found)
Schema history table [AutopilotDev].[dbo].[flyway_schema_history] does not exist yet

You are not signed in to Flyway, to sign in please run auth
Schema version: << Empty Schema >>

+----------+---------+-------------+--------------+--------------+---------+----------+
| Category | Version | Description | Type         | Installed On | State   | Undoable |
+----------+---------+-------------+--------------+--------------+---------+----------+
| Baseline | 001     | baseline    | SQL_BASELINE |              | Pending | No       |
+----------+---------+-------------+--------------+--------------+---------+----------+

# Specific environment
flyway info -environment=Test
flyway info -environment=Prod
```

Shows each migration, its version, description, state (`Pending` / `Success` / `Failed`), and applied timestamp.

---

### `migrate` — apply pending migrations

```powershell
# Default (development)
flyway migrate

# Target a specific environment
flyway migrate -environment=Test
flyway migrate -environment=Prod
```

Applies all `Pending` versioned (`V__`) and changed repeatable (`R__`) migrations in version order.

---

### `validate` — verify checksums match

```powershell
 ❯ flyway validate                                                     14:11:42
Flyway Community Edition 12.2.0 by Redgate

See release notes here: https://help.red-gate.com/help/flyway-cli12/help_2.aspx?topic=release-notes-and-older-versions/release-notes-for-flyway-engine
Database: jdbc:sqlserver://10.194.136.125:40000;connectRetryInterval=10;connectRetryCount=1;maxResultBuffer=-1;sendTemporalDataTypesAsStringForBulkCopy=true;delayLoadingLobs=true;useFmtOnly=false;cacheBulkCopyMetadata=false;bulkCopyForBatchInsertAllowEncryptedValueModifications=false;bulkCopyForBatchInsertTableLock=false;bulkCopyForBatchInsertKeepNulls=false;bulkCopyForBatchInsertKeepIdentity=false;bulkCopyForBatchInsertFireTriggers=false;bulkCopyForBatchInsertCheckConstraints=false;bulkCopyForBatchInsertBatchSize=0;useBulkCopyForBatchInsert=false;cancelQueryTimeout=-1;sslProtocol=TLS;calcBigDecimalPrecision=false;useDefaultJaasConfig=false;jaasConfigurationName=SQLJDBCDriver;statementPoolingCacheSize=0;serverPreparedStatementDiscardThreshold=10;enablePrepareOnFirstPreparedStatementCall=false;fips=false;socketTimeout=0;authentication=sqlPassword;authenticationScheme=nativeAuthentication;xopenStates=false;datetimeParameterType=datetime2;sendTimeAsDatetime=true;replication=false;trustStoreType=JKS;trustServerCertificate=true;TransparentNetworkIPResolution=true;iPAddressPreference=IPv4First;serverNameAsACE=false;sendStringParametersAsUnicode=true;selectMethod=direct;responseBuffering=adaptive;queryTimeout=-1;packetSize=8000;multiSubnetFailover=false;loginTimeout=30;lockTimeout=-1;lastUpdateCount=true;useDefaultGSSCredential=false;prepareMethod=prepexec;encrypt=mandatory;disableStatementPooling=true;databaseName=AutopilotDev;columnEncryptionSetting=Disabled;applicationName=Microsoft JDBC Driver for SQL Server;applicationIntent=readwrite; (Microsoft SQL Server 15.0)
ERROR: Skipping filesystem location: callbacks (not found)
Schema history table [AutopilotDev].[dbo].[flyway_schema_history] does not exist yet

You are not signed in to Flyway, to sign in please run auth
ERROR: Validate failed: Migrations have failed validation
Detected resolved migration not applied to database: 001.
To fix this error, either run migrate, or set -ignoreMigrationPatterns='*:pending'.
Need more flexibility with validation rules? Learn more: https://help.red-gate.com/help/flyway-cli12/help_2.aspx?topic=flyway-blog/older-posts/customize-validation-rules-with-ignoremigrationpatterns
flyway validate -environment=Test
```

Confirms that migration scripts on disk match what was applied to the database. Fails if a committed script was modified after being applied.

---

### `undo` — reverse the last applied migration

```powershell
flyway undo
flyway undo -environment=Test
```

Runs the corresponding `U{version}__*.sql` undo script. Requires Flyway Teams/Enterprise or the undo script to exist in `migrations/`.

---

### `clean` — wipe and rebuild (non-production only)

```powershell
flyway clean -environment=Build
flyway clean -environment=Check
```

Drops all objects in the target database. Only safe on environments with `provisioner = "clean"` (`Build`, `Check`). **Never run against `Test` or `Prod`.**

---

### `repair` — fix the schema history table

```powershell
flyway repair
flyway repair -environment=Test
```

Use when a migration is marked `Failed` in the history table after a partial run, or to realign checksums after an emergency fix applied directly to the DB.

---

## Full path (without PATH configured)

If you haven't added Flyway to PATH, prefix every command with the full path:

```powershell
D:\flyway-12.2.0\flyway info
D:\flyway-12.2.0\flyway migrate -environment=Test
D:\flyway-12.2.0\flyway validate -environment=Prod
```

---

## Typical dev-cycle sequence

```powershell
# 1. Check current state
flyway info

# 2. Apply your new migration (generated by Flyway Desktop)
flyway migrate

# 3. Confirm it applied cleanly
flyway info

# 4. Validate checksums still match
flyway validate
```

---

## Clean-build validation (CI pattern)

Used for the `Build` environment, which rebuilds from scratch on every run:

```powershell
# Apply fresh from baseline
flyway migrate -environment=Build

# Verify undo script round-trips
flyway undo -environment=Build
flyway migrate -environment=Build
```

---

## Related docs

- [flyway-pr-workflow.md](flyway-pr-workflow.md) — full GitHub PR process for schema changes
