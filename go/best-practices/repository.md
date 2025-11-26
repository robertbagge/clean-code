# Repository

Domain-scoped repository patterns with DB-to-domain mapping.

---

## Interface Definition

Define repository interfaces in the consumer package (service layer), returning
domain types—not database types:

```go
// internal/repository/interface.go
package repository

type PlanRequestRepository interface {
    GetPendingPlanRequests(ctx context.Context, limit int32) ([]domain.PlanRequest, error)
    GetPlanRequestByID(ctx context.Context, id string) (*domain.PlanRequest, error)
    UpdateStatus(ctx context.Context, id string, status domain.PlanRequestStatus, errMsg *string, errDetails map[string]any) error
    MarkStarted(ctx context.Context, id string) error
    MarkCompleted(ctx context.Context, id string) error
}

type TripRepository interface {
    GetTripWithJourneys(ctx context.Context, tripID string) (*domain.TripWithJourneys, error)
}
```

---

## Implementation Structure

```go
// internal/repository/plan_requests.go
package repository

// Compile-time interface verification
var _ PlanRequestRepository = (*PlanRequestRepo)(nil)

type PlanRequestRepo struct {
    queries *db.Queries
}

func NewPlanRequestRepo(dbtx db.DBTX) *PlanRequestRepo {
    return &PlanRequestRepo{
        queries: db.New(dbtx),
    }
}
```

The `var _ Interface = (*Impl)(nil)` pattern ensures the implementation
satisfies the interface at compile time.

---

## DB-to-Domain Mapping

Create mapping functions to convert sqlc-generated types to domain types:

```go
func (r *PlanRequestRepo) GetPendingPlanRequests(ctx context.Context, limit int32) ([]domain.PlanRequest, error) {
    rows, err := r.queries.GetPendingPlanRequests(ctx, limit)
    if err != nil {
        return nil, fmt.Errorf("get pending plan requests: %w", err)
    }

    requests := make([]domain.PlanRequest, 0, len(rows))
    for _, row := range rows {
        requests = append(requests, mapPlanRequest(row))
    }
    return requests, nil
}

// mapPlanRequest converts db.PlanRequest to domain.PlanRequest
func mapPlanRequest(row *db.PlanRequest) domain.PlanRequest {
    var errDetails map[string]any
    if row.ErrorDetails != nil {
        _ = json.Unmarshal(row.ErrorDetails, &errDetails)
    }

    return domain.PlanRequest{
        ID:           row.ID,
        TripID:       row.TripID,
        UserID:       row.UserID,
        Status:       domain.PlanRequestStatus(row.Status),
        ErrorMessage: row.ErrorMessage,
        ErrorDetails: errDetails,
        RetryCount:   deref(row.RetryCount),
        StartedAt:    row.StartedAt,
        CompletedAt:  row.CompletedAt,
        CreatedAt:    row.CreatedAt,
        UpdatedAt:    row.UpdatedAt,
    }
}
```

---

## Generic Dereference Helper

For nullable database columns mapped to pointers:

```go
// deref safely dereferences a pointer, returning zero value if nil
func deref[T any](p *T) T {
    if p == nil {
        var zero T
        return zero
    }
    return *p
}
```

Usage:

```go
RetryCount: deref(row.RetryCount),  // *int32 → int32
```

---

## Error Mapping

Map database errors to domain errors in the repository:

```go
func (r *PlanRequestRepo) GetPlanRequestByID(ctx context.Context, id string) (*domain.PlanRequest, error) {
    row, err := r.queries.GetPlanRequestByID(ctx, id)
    if err != nil {
        if err == pgx.ErrNoRows {
            return nil, domain.ErrPlanRequestNotFound
        }
        return nil, fmt.Errorf("get plan request %s: %w", id, err)
    }

    pr := mapPlanRequest(row)
    return &pr, nil
}
```

---

## Composed Queries

For complex data that spans multiple tables, compose queries in the repository:

```go
func (r *TripRepo) GetTripWithJourneys(ctx context.Context, tripID string) (*domain.TripWithJourneys, error) {
    // First query: get the trip
    tripRow, err := r.queries.GetTripByID(ctx, tripID)
    if err != nil {
        if err == pgx.ErrNoRows {
            return nil, domain.ErrTripNotFound
        }
        return nil, fmt.Errorf("get trip %s: %w", tripID, err)
    }

    // Second query: get related journeys
    journeyRows, err := r.queries.GetJourneysByTripID(ctx, tripID)
    if err != nil {
        return nil, fmt.Errorf("get journeys for trip %s: %w", tripID, err)
    }

    journeys := make([]domain.Journey, 0, len(journeyRows))
    for _, row := range journeyRows {
        journeys = append(journeys, mapJourney(row))
    }

    return &domain.TripWithJourneys{
        Trip:     mapTrip(tripRow),
        Journeys: journeys,
    }, nil
}
```

---

## Domain Types

Keep domain types clean and independent of database representation:

```go
// internal/domain/types.go
package domain

type PlanRequestStatus string

const (
    StatusPending   PlanRequestStatus = "pending"
    StatusStarted   PlanRequestStatus = "started"
    StatusCompleted PlanRequestStatus = "plan_generation_completed"
    StatusFailed    PlanRequestStatus = "failed"
)

type PlanRequest struct {
    ID           string
    TripID       string
    UserID       string
    Status       PlanRequestStatus  // Typed status, not raw string
    ErrorDetails map[string]any     // JSON deserialized in repository
    RetryCount   int32              // Not *int32—repository handles null
    StartedAt    *time.Time
    CompletedAt  *time.Time
    CreatedAt    time.Time
    UpdatedAt    time.Time
}
```

---

## Further Reading

* [database.md](./database.md) — sqlc configuration
* [error-handling.md](./error-handling.md) — error mapping patterns
* [package-structure.md](./package-structure.md) — where repositories fit
