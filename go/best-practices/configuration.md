# Configuration

Environment-based configuration patterns for Go services.

---

## Config Struct

Define a plain struct with explicit fields:

```go
// internal/config/config.go
package config

type Config struct {
    DatabaseURL     string
    LogLevel        string
    OpenAIAPIKey    string
    AnthropicAPIKey string
}
```

Avoid complex configuration frameworks. A simple struct with a loader function
is sufficient for most services.

---

## Loading Configuration

Load from environment variables with explicit validation:

```go
func LoadConfig() (*Config, error) {
    // Required variables
    dbURL := os.Getenv("DATABASE_URL")
    if dbURL == "" {
        return nil, fmt.Errorf("DATABASE_URL environment variable required")
    }

    openaiKey := os.Getenv("OPENAI_API_KEY")
    if openaiKey == "" {
        return nil, fmt.Errorf("OPENAI_API_KEY environment variable required")
    }

    // Optional variables (no error if missing)
    anthropicKey := os.Getenv("ANTHROPIC_API_KEY")
    logLevel := os.Getenv("LOG_LEVEL")

    return &Config{
        DatabaseURL:     dbURL,
        LogLevel:        logLevel,
        OpenAIAPIKey:    openaiKey,
        AnthropicAPIKey: anthropicKey,
    }, nil
}
```

### Guidelines

* **Required vars**: Return an error with a clear message if missing
* **Optional vars**: Use empty string or provide a sensible default
* **No defaults for secrets**: Always require explicit configuration

---

## Usage in main()

```go
func run() error {
    cfg, err := config.LoadConfig()
    if err != nil {
        return err
    }

    pool, err := config.NewDBPool(ctx, cfg.DatabaseURL)
    if err != nil {
        return err
    }
    defer pool.Close()

    // ... rest of setup
}
```

---

## Database Pool Configuration

Keep pool setup in the config package:

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

    if err := pool.Ping(ctx); err != nil {
        pool.Close()
        return nil, fmt.Errorf("pinging database: %w", err)
    }

    return pool, nil
}
```

---

## Cloud Run Settings

For Cloud Run with Supavisor (or similar poolers):

| Setting | Value | Why |
|---------|-------|-----|
| `QueryExecModeSimpleProtocol` | Required | Transaction mode poolers don't support prepared statements |
| `MaxConns = 15` | Moderate | Cloud Run instances share pooler connections |
| `MinConns = 3` | Low | Balance warmth vs resource usage |
| `MaxConnLifetime = 1h` | Standard | Rotate connections periodically |
| `MaxConnIdleTime = 30m` | Moderate | Release unused connections |
| `HealthCheckPeriod = 1m` | Standard | Detect dead connections |

---

## Avoid

* **Framework overkill**: Viper, envconfig, etc. add complexity without value
  for simple services
* **Defaulting secrets**: Never default API keys or database URLs
* **Global config**: Pass config through constructors, don't use package globals
* **Lazy loading**: Load all config at startup; fail fast on missing values

---

## Further Reading

* [go-style.md](./go-style.md) — prefer concrete types for configuration
* [database.md](./database.md) — connection pool details
* [KISS](../clean-code/kiss.md) — simplest working design
