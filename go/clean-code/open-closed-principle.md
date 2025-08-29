# Open/Closed Principle (OCP) in Go

## Overview

OCP: **open for extension, closed for modification**. In Go, you get this
by leaning on **interfaces, composition, and registration**. Add new behavior
by *adding new implementations*, not by editing existing business logic.

> Pragmatic note: If you keep **SRP, OCP, ISP, DIP** tight, you're \~90% of
> the way to good Go design. LSP tends to "fall out" when your interfaces are
> small and consumer-defined.

---

## Core Concept

* **Define behavior as interfaces** in the *consumer* package.
* **Extend via new implementations**, not by changing call sites.
* Prefer **composition** (and optional embedding) over edits.
* Consider a **registration/factory** for pluggable backends.
* Keep interfaces **small** (pairs nicely with ISP).

---

## Scaffolding (so snippets compile)

```go
package notify

import (
    "context"
    "fmt"
)
```

---

## BAD — Switch explosion (violates OCP)

```go
func SendNotification(ctx context.Context, message, channel string) error {
    switch channel {
    case "email":
        // send email
    case "sms":
        // send sms
    // adding "slack", "teams", "webhook" requires editing this switch
    default:
        return fmt.Errorf("unknown channel: %s", channel)
    }
    return nil
}
```

Adding a new channel forces edits to existing code (risk, review churn, redeploy).

---

## GOOD — Interface + composition (OCP-friendly)

```go
// Define behavior in the consumer package.
type Notifier interface {
    Send(ctx context.Context, message string) error
}

// Implementations (live in same or subpackages; no need to change consumers).
type EmailNotifier struct{ SMTPHost string }
func (e *EmailNotifier) Send(ctx context.Context, message string) error {
    // ... send email ...
    return nil
}

type SMSNotifier struct{ APIKey string }
func (s *SMSNotifier) Send(ctx context.Context, message string) error {
    // ... send sms ...
    return nil
}

type SlackNotifier struct{ WebhookURL string }
func (s *SlackNotifier) Send(ctx context.Context, message string) error {
    // ... post to Slack ...
    return nil
}

// Service composes Notifiers; open to extension by adding more implementations.
type NotificationService struct {
    notifiers []Notifier
}

func NewNotificationService(ns ...Notifier) *NotificationService {
    return &NotificationService{notifiers: ns}
}

func (n *NotificationService) Add(notifier Notifier) {
    n.notifiers = append(n.notifiers, notifier)
}

func (n *NotificationService) Broadcast(ctx context.Context, msg string) {
    for _, no := range n.notifiers {
        if err := no.Send(ctx, msg); err != nil {
            fmt.Printf("notify failed: %v\n", err) // log and continue
        }
    }
}
```

To add Slack/Teams/Webhook, *add a new type that satisfies `Notifier`*; no
edits to `NotificationService`.

---

## Optional: Plugin registration (extensible without touching wiring)

```go
// Simple registry for runtime-configurable backends.
type Factory func(cfg map[string]string) (Notifier, error)

var registry = map[string]Factory{}

func Register(name string, f Factory) { registry[name] = f }

func BuildFromConfig(specs []struct {
    Name string
    Cfg  map[string]string
}) (*NotificationService, error) {
    var impls []Notifier
    for _, s := range specs {
        f, ok := registry[s.Name]
        if !ok { return nil, fmt.Errorf("unknown notifier: %s", s.Name) }
        n, err := f(s.Cfg); if err != nil { return nil, err }
        impls = append(impls, n)
    }
    return NewNotificationService(impls...), nil
}

// Register implementations (in their init or package-level func).
func init() {
    Register("email", func(cfg map[string]string) (Notifier, error) {
        return &EmailNotifier{SMTPHost: cfg["smtp_host"]}, nil
    })
    Register("sms", func(cfg map[string]string) (Notifier, error) {
        return &SMSNotifier{APIKey: cfg["api_key"]}, nil
    })
    Register("slack", func(cfg map[string]string) (Notifier, error) {
        return &SlackNotifier{WebhookURL: cfg["webhook_url"]}, nil
    })
}
```

Now new channels are **just new registrations**.

---

## Testing OCP (lightweight)

```go
// Fake to verify service behavior without real backends.
type FakeNotifier struct{ msgs []string; fail bool }
func (f *FakeNotifier) Send(ctx context.Context, m string) error {
    if f.fail { return fmt.Errorf("boom") }
    f.msgs = append(f.msgs, m); return nil
}

// Example test idea:
// - Create service with FakeNotifier(s)
// - Broadcast; assert f.msgs contains message
// - Add another Notifier; rerun without changing service code
```

---

## Anti-patterns to Avoid

1. **Switch/type-assertion cascades** in consumers for behavior.
2. **Modifying existing code** to add a backend/algorithm.
3. **Provider-owned fat interfaces** (define interfaces in consumers, keep them small).

---

## Key Takeaways

* Model behavior as **small consumer-defined interfaces**.
* Extend by **adding implementations**, not editing call sites.
* Use **composition** and (optionally) a **registry** to plug new behavior.
* Test via fakes; consumers shouldn’t change when implementations change.

---

## Related Best Practices

For package structure, where to define interfaces, error placement, and
testing patterns (fakes, table-driven tests, golden files), see
👉 **[best-practices.md](../best-practices.md)**
