# Flyway Desktop + GitHub PR Workflow

This document describes the end-to-end process for making database schema changes in this project using Flyway Desktop and the GitHub pull-request review process.

---

## Repository structure

```
.
├── flyway.toml              # Shared project config (committed)
├── flyway.user.toml         # Local credentials (gitignored)
├── Filter.scpf              # Redgate SQL Compare object filter
├── migrations/              # Versioned migration scripts (V/U/R prefix)
│   └── B001__baseline.sql   # Baseline — every environment starts here
├── schema-model/            # Source-of-truth schema as individual .sql files
│   ├── Security/Schemas/
│   ├── Stored Procedures/
│   ├── Tables/
│   └── Views/
└── Scripts/                 # Utility scripts (not run by Flyway)
```

---

## Environments

| Name | Database | Purpose | Provisioner |
|---|---|---|---|
| `development` | AutopilotDev | Developer's working DB — you change this directly | — |
| `shadow` | AutopilotShadow | Flyway Desktop's internal reference DB | `clean` |
| `Build` | AutopilotBuild | CI validation — clean build on every run | `clean` |
| `Check` | AutopilotCheck | Reporting / QA environment | `clean` |
| `Test` | AutopilotTest | Pre-production testing | — |
| `Prod` | AutopilotProd | Production | — |

`development` and `shadow` are in `flyway.user.toml` (gitignored). All others are in `flyway.toml` with credentials supplied via `flyway.user.toml`.

---

## How Flyway Desktop tracks changes

Flyway Desktop keeps two things in sync:

1. **`schema-model/`** — individual `.sql` files representing the current desired schema state (one file per object)
2. **`migrations/`** — ordered, versioned scripts that transform the database from one state to the next

The `shadow` database holds the schema that results from running all existing migrations. When you change the `development` database, Flyway Desktop compares it against `shadow` to detect the delta, then generates a new migration script to capture it.

---

## Step-by-step workflow

### 1. Create a feature branch

```bash
git checkout main
git pull
git checkout -b feature/your-change-description
```

### 2. Make schema changes in Flyway Desktop

1. Open Flyway Desktop and open this project.
2. Confirm the active environment is **development**.
3. Make your DDL changes directly on **AutopilotDev** using SSMS or Azure Data Studio.
4. Return to Flyway Desktop — it detects the diff between `development` and `shadow`.

### 3. Generate a migration script

1. In the **Migrations** tab, click **Generate migration**.
2. Flyway Desktop will:
   - Create a new versioned file — e.g. `migrations/V002__add_loyalty_tier_column.sql`
   - Create a corresponding undo script — e.g. `migrations/U002__add_loyalty_tier_column.sql`
   - Update the affected `schema-model/` files to reflect the new state
3. Review and adjust the generated SQL if needed (e.g. backfill data, handle nullable constraints).

> **Naming rules** (`validateMigrationNaming = true`)
> - `V{version}__{description}.sql` — versioned migration
> - `U{version}__{description}.sql` — undo script
> - `R__{description}.sql` — repeatable (stored procedures, views)
> - Version numbers must be unique and higher than all existing ones.

### 4. Commit and push

```bash
git add migrations/V002__add_loyalty_tier_column.sql
git add migrations/U002__add_loyalty_tier_column.sql
git add schema-model/Tables/Sales.LoyaltyProgram.sql

git commit -m "feat: add loyalty_tier column to Sales.LoyaltyProgram"
git push origin feature/add-loyalty-tier-column
```

> Do **not** commit `flyway.user.toml` — it is gitignored and contains local credentials.

### 5. Open a Pull Request

Target `main`. Use this PR description template:

```
## What changed
Brief description of the schema change.

## Migration scripts
- migrations/V002__add_loyalty_tier_column.sql
- migrations/U002__add_loyalty_tier_column.sql (undo)

## Schema model files changed
- schema-model/Tables/Sales.LoyaltyProgram.sql

## Testing
- [ ] Migration validated against Build environment
- [ ] Undo script tested and reverses cleanly
- [ ] No data loss for existing rows
```

### 6. PR review checklist

**Migration script (`V*.sql`)**
- [ ] Safe to run on a live database — no unconditional `DROP`, data-safe `ALTER`
- [ ] Handles existing rows correctly (default values for new NOT NULL columns)
- [ ] Version number does not conflict with existing scripts

**Undo script (`U*.sql`)**
- [ ] Correctly reverses the versioned migration
- [ ] Tested against the `development` environment

**Schema model (`schema-model/**`)**
- [ ] Only the intended objects are changed
- [ ] No credentials or environment-specific values committed

### 7. CI validation against Build

The `Build` environment uses `provisioner = "clean"` — it rebuilds from scratch on every run, making it perfect for CI validation:

```bash
flyway migrate -environment=Build
```

To also verify the undo script round-trips cleanly:

```bash
flyway undo -environment=Build
flyway migrate -environment=Build
```

A failure here must be resolved before merge.

### 8. Merge and deploy

Once the PR is approved and merged to `main`:

```bash
# Pre-production
flyway migrate -environment=Test

# Production
flyway migrate -environment=Prod
```

`outOfOrder = true` means migrations applied out of sequence (e.g. hotfix branches) will still be accepted.

---

## Repeatable migrations (stored procedures & views)

Objects under `schema-model/Stored Procedures/` and `schema-model/Views/` are re-runnable. Use **Repeatable migrations** (`R__`) instead of versioned `V__` scripts:

```
migrations/R__Customers.RecordFeedback.sql
migrations/R__Sales.CustomerOrdersView.sql
```

Flyway re-runs these whenever their checksum changes. Copy the full `CREATE OR ALTER` definition from the schema-model file into the `R__` migration file.

---

## Rollback

```bash
flyway undo -environment=Test
```

Requires a corresponding `U{version}__` undo file. The `Build` and `Check` environments support full clean + re-migrate as an alternative.

---

## Key files reference

| File | Tracked | Purpose |
|---|---|---|
| `flyway.toml` | Yes | Project config, environment URLs, comparison options |
| `flyway.user.toml` | **No** | Local credentials (gitignored) |
| `Filter.scpf` | Yes | Controls which SQL object types are compared |
| `migrations/` | Yes | All versioned migration scripts |
| `schema-model/` | Yes | Source-of-truth schema (updated by Flyway Desktop) |