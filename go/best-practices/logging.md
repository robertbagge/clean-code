# Logging

Structured logging patterns using Go's standard `log/slog` package.

---

## Setup

Configure a JSON handler with source location tracking:

```go
import (
    "log/slog"
    "os"
)

func setupLogger() {
    handler := slog.NewJSONHandler(os.Stderr, &slog.HandlerOptions{
        AddSource: true,           // Include file:line in output
        Level:     slog.LevelInfo,
    })
    logger := slog.New(handler)
    slog.SetDefault(logger)
}
```

Call `setupLogger()` early in `main()` before any logging occurs.

---

## Structured Attributes

Use typed attribute functions for structured data:

```go
// Basic info logging
slog.Info("connected to database")

// With structured attributes
slog.Info("found pending requests", slog.Int("count", len(requests)))

// Multiple attributes
slog.Info("pipeline completed",
    slog.String("request_id", requestID),
    slog.Int64("duration_ms", time.Since(start).Milliseconds()),
    slog.Int("plan_day_count", len(result.Plan)))
```

### Attribute Functions

| Function | Use Case |
|----------|----------|
| `slog.String(key, val)` | String values |
| `slog.Int(key, val)` | Integer values |
| `slog.Int64(key, val)` | Large integers (e.g., milliseconds) |
| `slog.Bool(key, val)` | Boolean flags |
| `slog.Any(key, val)` | Complex types, errors |
| `slog.Duration(key, val)` | Time durations |

---

## Error Logging

Always include the error using `slog.Any`:

```go
if err != nil {
    slog.Error("failed to process request",
        slog.String("request_id", req.ID),
        slog.Any("error", err))
    return err
}
```

---

## Contextual Logging

Create child loggers with persistent attributes for request-scoped logging:

```go
func (s *Service) ProcessRequest(ctx context.Context, requestID string) error {
    logger := s.logger.With(slog.String("request_id", requestID))

    logger.Info("starting processing")

    // ... do work ...

    logger.Info("processing completed",
        slog.Int64("duration_ms", time.Since(start).Milliseconds()))
    return nil
}
```

---

## Dependency Injection

Pass loggers through constructors rather than using the global default:

```go
type PipelineService struct {
    repo   Repository
    logger *slog.Logger
}

func NewPipelineService(repo Repository, logger *slog.Logger) *PipelineService {
    return &PipelineService{
        repo:   repo,
        logger: logger,
    }
}
```

In `main()`, pass the default logger:

```go
logger := slog.Default()
svc := NewPipelineService(repo, logger)
```

---

## Signal Handling

Log shutdown signals for observability:

```go
func handleShutdown(cancel context.CancelFunc) {
    sigCh := make(chan os.Signal, 1)
    signal.Notify(sigCh, syscall.SIGINT, syscall.SIGTERM)
    sig := <-sigCh
    slog.Info("received shutdown signal", slog.String("signal", sig.String()))
    cancel()
}
```

---

## Output Format

With `NewJSONHandler` and `AddSource: true`, logs look like:

```json
{
  "time": "2025-01-15T10:30:00Z",
  "level": "INFO",
  "source": {"function": "main.run", "file": "main.go", "line": 47},
  "msg": "connected to database"
}
```

---

## Further Reading

* [slog package documentation](https://pkg.go.dev/log/slog)
* [configuration.md](./configuration.md) — environment-based log level config
