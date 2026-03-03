# Step 02.00 — Platform metadata schema (PostgreSQL)

This migration defines **platform metadata** tables required by the centralized data service:

- Users & RBAC (roles + bindings)
- Audit logs (security/admin actions)
- Query history and daily metrics (performance/monitoring)
- Dynamic schema management metadata (managed schemas/tables/columns)
- A simple metadata KV store for platform config/state
- A lightweight `schema_migrations` ledger (works naturally with `pg_dump`/`psql` restore)

> Note: The actual DDL was applied directly to PostgreSQL (per container rules: execute statements one-at-a-time using `psql -c` and connection from `db_connection.txt`).  
> This file serves as the migration artifact/documentation for what now exists in the database.

---

## Connection source (container rule)

Use the connection string from:

- `centralized_db_database/db_connection.txt` (format: `psql postgresql://...`)

---

## Extension

- `pgcrypto` is enabled to support `gen_random_uuid()` for UUID primary keys:

```sql
CREATE EXTENSION IF NOT EXISTS pgcrypto;
```

---

## Core helper

### updated_at trigger function

```sql
CREATE OR REPLACE FUNCTION set_updated_at()
RETURNS trigger
LANGUAGE plpgsql
AS $$
BEGIN
  NEW.updated_at = now();
  RETURN NEW;
END;
$$;
```

Used by triggers on tables that contain an `updated_at` column.

---

## Auth / Users / RBAC

### `app_users`

Purpose: application user accounts (auth).

Key points:
- UUID PK (`gen_random_uuid()`)
- `email` unique
- `password_hash` stored (bcrypt/argon etc. handled by backend)
- `updated_at` maintained via trigger

```sql
CREATE TABLE IF NOT EXISTS app_users (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  email text NOT NULL UNIQUE,
  password_hash text NOT NULL,
  display_name text,
  is_active boolean NOT NULL DEFAULT true,
  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now(),
  last_login_at timestamptz
);

DROP TRIGGER IF EXISTS trg_app_users_updated_at ON app_users;
CREATE TRIGGER trg_app_users_updated_at
BEFORE UPDATE ON app_users
FOR EACH ROW EXECUTE FUNCTION set_updated_at();
```

### `roles`

Purpose: RBAC role definitions (Admin/Developer/Viewer, etc).

```sql
CREATE TABLE IF NOT EXISTS roles (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  name text NOT NULL UNIQUE,
  description text,
  created_at timestamptz NOT NULL DEFAULT now()
);
```

### `user_role_bindings`

Purpose: user-to-role bindings.

Key points:
- Composite PK `(user_id, role_id)` prevents duplicates
- Cascades on user/role deletion
- Tracks `granted_by` and `granted_at`

```sql
CREATE TABLE IF NOT EXISTS user_role_bindings (
  user_id uuid NOT NULL REFERENCES app_users(id) ON DELETE CASCADE,
  role_id uuid NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
  granted_by uuid REFERENCES app_users(id) ON DELETE SET NULL,
  granted_at timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (user_id, role_id)
);
```

---

## Audit logs

### `audit_logs`

Purpose: immutable-ish record of actions for compliance/security.

```sql
CREATE TABLE IF NOT EXISTS audit_logs (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  actor_user_id uuid REFERENCES app_users(id) ON DELETE SET NULL,
  action text NOT NULL,
  resource_type text NOT NULL,
  resource_id text,
  success boolean NOT NULL DEFAULT true,
  ip inet,
  user_agent text,
  request_id text,
  metadata jsonb NOT NULL DEFAULT '{}'::jsonb,
  created_at timestamptz NOT NULL DEFAULT now()
);
```

Indexes:

```sql
CREATE INDEX IF NOT EXISTS idx_audit_logs_created_at
  ON audit_logs (created_at DESC);

CREATE INDEX IF NOT EXISTS idx_audit_logs_actor_created_at
  ON audit_logs (actor_user_id, created_at DESC);

CREATE INDEX IF NOT EXISTS idx_audit_logs_resource_created_at
  ON audit_logs (resource_type, resource_id, created_at DESC);
```

---

## Query history & metrics

### `query_history`

Purpose: store validated query executions (for admin UI, debugging, performance).

```sql
CREATE TABLE IF NOT EXISTS query_history (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  executed_by uuid REFERENCES app_users(id) ON DELETE SET NULL,
  request_id text,
  query_text text NOT NULL,
  query_hash text NOT NULL,
  query_type text,
  target_schema text,
  target_table text,
  parameters jsonb NOT NULL DEFAULT '{}'::jsonb,
  success boolean NOT NULL DEFAULT true,
  error_message text,
  row_count integer,
  duration_ms integer,
  started_at timestamptz NOT NULL DEFAULT now(),
  finished_at timestamptz,
  created_at timestamptz NOT NULL DEFAULT now()
);
```

Indexes:

```sql
CREATE INDEX IF NOT EXISTS idx_query_history_created_at
  ON query_history (created_at DESC);

CREATE INDEX IF NOT EXISTS idx_query_history_executed_by_created_at
  ON query_history (executed_by, created_at DESC);

CREATE INDEX IF NOT EXISTS idx_query_history_hash_created_at
  ON query_history (query_hash, created_at DESC);

CREATE INDEX IF NOT EXISTS idx_query_history_target_created_at
  ON query_history (target_schema, target_table, created_at DESC);
```

### `query_metrics_daily`

Purpose: aggregated metrics for dashboards (daily rollups).

```sql
CREATE TABLE IF NOT EXISTS query_metrics_daily (
  day date NOT NULL,
  query_hash text NOT NULL,
  query_type text,
  target_schema text,
  target_table text,
  exec_count bigint NOT NULL DEFAULT 0,
  success_count bigint NOT NULL DEFAULT 0,
  error_count bigint NOT NULL DEFAULT 0,
  total_duration_ms bigint NOT NULL DEFAULT 0,
  min_duration_ms integer,
  max_duration_ms integer,
  p95_duration_ms integer,
  avg_duration_ms integer,
  PRIMARY KEY (day, query_hash)
);

CREATE INDEX IF NOT EXISTS idx_query_metrics_daily_day
  ON query_metrics_daily (day DESC);
```

---

## Dynamic schema management metadata

These tables allow the backend/dashboard to track which schemas/tables/columns are managed by the platform (in addition to PostgreSQL system catalogs).

### `managed_schemas`

```sql
CREATE TABLE IF NOT EXISTS managed_schemas (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  schema_name text NOT NULL UNIQUE,
  description text,
  created_by uuid REFERENCES app_users(id) ON DELETE SET NULL,
  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now()
);

DROP TRIGGER IF EXISTS trg_managed_schemas_updated_at ON managed_schemas;
CREATE TRIGGER trg_managed_schemas_updated_at
BEFORE UPDATE ON managed_schemas
FOR EACH ROW EXECUTE FUNCTION set_updated_at();
```

### `managed_tables`

```sql
CREATE TABLE IF NOT EXISTS managed_tables (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  schema_id uuid NOT NULL REFERENCES managed_schemas(id) ON DELETE CASCADE,
  table_name text NOT NULL,
  display_name text,
  description text,
  created_by uuid REFERENCES app_users(id) ON DELETE SET NULL,
  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now(),
  UNIQUE (schema_id, table_name)
);

DROP TRIGGER IF EXISTS trg_managed_tables_updated_at ON managed_tables;
CREATE TRIGGER trg_managed_tables_updated_at
BEFORE UPDATE ON managed_tables
FOR EACH ROW EXECUTE FUNCTION set_updated_at();

CREATE INDEX IF NOT EXISTS idx_managed_tables_schema_id
  ON managed_tables (schema_id);
```

### `managed_columns`

```sql
CREATE TABLE IF NOT EXISTS managed_columns (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  table_id uuid NOT NULL REFERENCES managed_tables(id) ON DELETE CASCADE,
  column_name text NOT NULL,
  data_type text NOT NULL,
  is_nullable boolean NOT NULL DEFAULT true,
  is_primary_key boolean NOT NULL DEFAULT false,
  default_value text,
  ordinal_position integer,
  description text,
  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now(),
  UNIQUE (table_id, column_name)
);

DROP TRIGGER IF EXISTS trg_managed_columns_updated_at ON managed_columns;
CREATE TRIGGER trg_managed_columns_updated_at
BEFORE UPDATE ON managed_columns
FOR EACH ROW EXECUTE FUNCTION set_updated_at();

CREATE INDEX IF NOT EXISTS idx_managed_columns_table_id
  ON managed_columns (table_id);
```

---

## Platform metadata KV

### `metadata_kv`

Purpose: store platform-wide config/state blobs (feature flags, last processed event cursor, etc).

```sql
CREATE TABLE IF NOT EXISTS metadata_kv (
  key text PRIMARY KEY,
  value jsonb NOT NULL,
  updated_at timestamptz NOT NULL DEFAULT now()
);
```

---

## Migration ledger (backup/restore-aligned)

### `schema_migrations`

Purpose: track applied migrations in-db so `pg_dump` / `psql` restore includes the ledger automatically.

```sql
CREATE TABLE IF NOT EXISTS schema_migrations (
  id bigserial PRIMARY KEY,
  migration_name text NOT NULL UNIQUE,
  applied_at timestamptz NOT NULL DEFAULT now()
);
```

Recorded entry:

```sql
INSERT INTO schema_migrations (migration_name)
VALUES ('02_00_platform_metadata')
ON CONFLICT (migration_name) DO NOTHING;
```

---

## Notes on backup/restore alignment

The container’s `backup_db.sh` uses:

- `pg_dump --clean --if-exists --create > database_backup.sql`

and restore uses:

- `psql ... -d postgres < database_backup.sql`

Because all objects above are standard PostgreSQL DDL in the target database:
- They are included in `pg_dump`
- They are dropped/recreated by the `--clean --if-exists` behavior
- The `schema_migrations` ledger is restored as well (so the system knows what was applied)
