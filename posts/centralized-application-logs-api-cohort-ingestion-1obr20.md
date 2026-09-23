# Centralized Application Logs API — Cohort Ingestion and Search With Tenant Attribution

Short answer: choose an ingestion and search API only after proving that each edtech experiment event retains a stable tenant identifier, cohort assignment, event identity, and timestamp from emission through retrieval. For a startup dashboard, an application-owned, structured event contract feeding a centralized store with bounded search makes cost attribution possible at ingress. A search box alone cannot establish whether a retried enrollment event was counted twice or whether a query exposed another school's records.

## Which API should a startup use for centralized application logs ingestion and search?

The decision is to emit structured logs at the application boundary, ingest them asynchronously, and expose tenant-scoped search to the dashboard. Treat the store as diagnostic evidence, not as the authoritative ledger for experiment outcomes or billing. Comparing treatment and control cohorts requires an explicit assignment identifier and versioned experiment identifier: deriving membership from message text, sampled traces, or a dashboard's current filter creates a moving denominator.

Event identity matters. Retries happen.

Assign the same logical event the same ID across delivery attempts, while distinguishing it from a second actual student action. Preserve the original occurrence time and record ingestion time separately; delayed delivery then remains visible without rewriting the sequence. Bind tenant, experiment, cohort, event ID, and outcome before ingestion. Trace context can connect diagnostics across services, but a trace ID does not establish tenant authorization or durable business identity. In a two-cohort experiment, an event without an assignment version has an ambiguous denominator even if the log message says "treatment"; it should be quarantined from outcome calculations until the source of that assignment is established. The same rule applies to a delayed record whose event time falls before reassignment but whose intake time falls after it.

Enforce the tenant filter on the server, rather than accepting a browser-provided scope. Exclude student identifiers and free-form payloads where possible, and decide access, retention, and deletion policy before persisting records. OWASP's logging guidance addresses minimization and protection; a log pipeline alone cannot certify compliance with a jurisdiction or school contract. Those obligations require review of the actual data, retention schedule, and processor arrangements.

## How should ingestion and search divide responsibility?

The dashboard calls an authenticated search API with server-derived tenant scope, a time range, experiment version, cohort, cursor, and bounded limit. The backend sends structured records through durable delivery, so an ingestion retry preserves event identity. Search indexes serve investigation; cohort metrics come from deduplicated authoritative events or an explicitly documented aggregation pipeline. Otherwise, delayed indexing could silently change a published comparison.

| Design | Useful boundary | Attribution and failure boundary |
| --- | --- | --- |
| Structured intake plus tenant-scoped search | Stable event schema across services | Meter accepted usage by authenticated tenant; observe queue and index lag separately. |
| Direct writes to searchable storage | Small deployments with limited write pressure | Fewer moving parts, but storage outages can reach the request path unless writes are isolated. |
| Search over unstructured text | Temporary, low-risk debugging | Parsing drift weakens cohort grouping and makes tenant attribution hard to audit. |

An intake receipt and an index watermark represent different promises. Acceptance means the intake boundary has recorded an event, not that it is searchable or that a cohort metric is final. Count rejected records by reason, measure queue age and search delay, and reconcile accepted IDs against indexed IDs over a defined window. For cost attribution, meter accepted bytes against the authenticated tenant before fan-out or sampling, then reconcile that meter with storage and query usage. Shared infrastructure and retention overhead need an explicit allocation rule, rather than a claim of exact per-tenant cost.

The distinction is operational.

## What belongs on the critical path?

This Go sketch puts the identity and scope checks at intake. An authenticated context supplies the tenant; transport authentication, durable publication, and storage implementation remain separate concerns. Rejecting a mismatch prevents client-supplied tenant IDs from corrupting both access control and usage attribution.

```go
package logging

import (
    "context"
    "errors"
    "time"
)

type Event struct {
    ID         string
    TenantID   string
    Experiment string
    Cohort     string
    OccurredAt time.Time
    Outcome    string
}

type Publisher interface {
    Publish(context.Context, Event) error
}

type Meter interface {
    RecordAccepted(context.Context, string, string, int) error
}

func Accept(ctx context.Context, tenant string, e Event, size int, pub Publisher, meter Meter) error {
    if tenant == "" || e.TenantID != tenant || e.ID == "" ||
        e.Experiment == "" || e.Cohort == "" || e.OccurredAt.IsZero() || size < 0 {
        return errors.New("invalid event identity or tenant scope")
    }
    if err := pub.Publish(ctx, e); err != nil {
        return err
    }
    return meter.RecordAccepted(ctx, tenant, e.ID, size)
}
```

This is an interface sketch, not a transactional guarantee. If publication succeeds and metering fails, a retry must not double-count usage: implement both through one durable intake record or a transactional outbox, with metering deduplicated by tenant and event ID. A returned error may follow a successful publication. Test ambiguous outcomes, repeated IDs, out-of-order delivery, missing assignment versions, and attempts to search another tenant. The read path must authorize before constructing its search predicate, cap the time window and page size, and display index lag beside results.

No receipt implies search freshness.

Deploy schema and reader changes compatibly: accept the old event shape during transition, validate new cohort fields at ingress, then tighten validation after producers change. Keep raw diagnostic records out of the experiment's statistical denominator until the assignment and event semantics have been reviewed. An ingestion pipeline can be healthy while the comparison is wrong.

## Why reject unstructured text as the default?

Free-text aggregation is valid for short-lived debugging in a bounded, low-sensitivity environment where investigators need error discovery, not tenant-level charges or cohort conclusions. It is a poor default here: a message-format change can break grouping, and a missing tenant field may be impossible to reconstruct after ingestion. The schema work pays for an auditable comparison and defensible allocation, not a prettier interface.

The limitation of structured intake is the work of maintaining a versioned event contract, durable intake, deduplication, and server-side authorization. The trade-off is justified only when attribution or cohort comparisons depend on those guarantees. This approach is not suitable for a small team whose only need is temporary internal error lookup; its existing scoped log store is enough until cohort accounting actually becomes a requirement. For this decision, validate duplicate delivery and delayed indexing with a replay test, test cross-tenant authorization, review retention and deletion, and reconcile accepted usage against indexed records. Search convenience follows proof of those boundaries.

## References

- OpenTelemetry Logs Data Model: https://opentelemetry.io/docs/specs/otel/logs/data-model/
- W3C Trace Context: https://www.w3.org/TR/trace-context/
- OWASP Logging Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html
- GDPR, Article 5 (principles relating to processing of personal data): https://eur-lex.europa.eu/eli/reg/2016/679/oj/eng
