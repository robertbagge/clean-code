# Database

Type-safe database access using sqlc and pgx.

---

## sqlc Configuration

sqlc generates type-safe Go code from SQL queries. Configure it in `sqlc.yaml`:

```yaml
version: "2"
sql:
  - engine: "postgresql"
    queries: "queries/"
    schema: "path/to/migrations/"
    gen:
      go:
        package: "db"
        out: "internal/db"
        sql_package: "pgx/v5"
        emit_json_tags: true
        emit_prepared_queries: false  # CRITICAL for Supavisor transaction mode
        emit_interface: true
        emit_result_struct_pointers: true
        emit_params_struct_pointers: true
```

### Critical: Prepared Queries

Set `emit_prepared_queries: false` when using connection poolers like Supavisor
in transaction mode. Prepared statements don't work across pooled connections.

---

## Type Mappings

Configure explicit type mappings for common PostgreSQL types:

```yaml
overrides:
  # UUID → string (pgx handles conversion automatically)
  - db_type: "uuid"
    go_type: "string"
  - db_type: "uuid"
    nullable: true
    go_type:
      type: "string"
      pointer: true

  # timestamptz → time.Time
  - db_type: "timestamptz"
    go_type:
      import: "time"
      type: "Time"
  - db_type: "timestamptz"
    nullable: true
    go_type:
      import: "time"
      type: "Time"
      pointer: true

  # Nullable text → *string
  - db_type: "text"
    nullable: true
    go_type:
      type: "string"
      pointer: true

  # Nullable integers → *int32
  - db_type: "int4"
    nullable: true
    go_type:
      type: "int32"
      pointer: true

  # jsonb → defaults to []byte with pgx/v5
```

---

## Query Naming Conventions

Use sqlc's naming comments to specify return types:

```sql
-- name: GetUserByID :one
-- Returns a single row
SELECT id, email, name FROM users WHERE id = $1;

-- name: GetActiveUsers :many
-- Returns multiple rows
SELECT id, email, name FROM users WHERE active = true;

-- name: UpdateUserEmail :exec
-- Returns nothing (just executes)
UPDATE users SET email = $2 WHERE id = $1;

-- name: CreateUser :one
-- Returns the created row
INSERT INTO users (email, name) VALUES ($1, $2) RETURNING *;
```

| Suffix | Return Type |
|--------|-------------|
| `:one` | Single struct pointer |
| `:many` | Slice of structs |
| `:exec` | No return (error only) |
| `:execrows` | Number of affected rows |

---

## Connection Pool Configuration

Configure pgx pool for cloud environments (e.g., Cloud Run with Supavisor):

```go
func NewDBPool(ctx context.Context, databaseURL string) (*pgxpool.Pool, error) {
    config, err := pgxpool.ParseConfig(databaseURL)
    if err != nil {
        return nil, fmt.Errorf("parsing database URL: %w", err)
    }

    // CRITICAL: Required for Supavisor transaction mode
    config.ConnConfig.DefaultQueryExecMode = pgx.QueryExecModeSimpleProtocol

    // Connection pool settings optimized for Cloud Run
    config.MaxConns = 15
    config.MinConns = 3
    config.MaxConnLifetime = time.Hour
    config.MaxConnIdleTime = 30 * time.Minute
    config.HealthCheckPeriod = time.Minute

    pool, err := pgxpool.NewWithConfig(ctx, config)
    if err != nil {
        return nil, fmt.Errorf("creating connection pool: %w", err)
    }

    // Verify connection
    if err := pool.Ping(ctx); err != nil {
        pool.Close()
        return nil, fmt.Errorf("pinging database: %w", err)
    }

    return pool, nil
}
```

### Pool Settings

| Setting | Value | Rationale |
|---------|-------|-----------|
| `MaxConns` | 15 | Cloud Run instances share pooler connections |
| `MinConns` | 3 | Keep warm connections for latency |
| `MaxConnLifetime` | 1 hour | Rotate connections periodically |
| `MaxConnIdleTime` | 30 min | Release unused connections |
| `HealthCheckPeriod` | 1 min | Detect dead connections |

---

## Usage Pattern

```go
// Initialize pool
pool, err := NewDBPool(ctx, cfg.DatabaseURL)
if err != nil {
    return err
}
defer pool.Close()

// Create repository with pool
repo := repository.NewUserRepo(pool)
```

---

## Further Reading

* [sqlc documentation](https://docs.sqlc.dev/)
* [pgx documentation](https://github.com/jackc/pgx)
* [repository.md](./repository.md) — repository patterns using sqlc
