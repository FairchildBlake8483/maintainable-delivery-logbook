# An Audit Ledger for Daily, Weekly Node.js Reminders: Timezone-Aware DST Handling

Short answer: represent each daily or weekly reminder as a local-time rule plus an IANA timezone, resolve upcoming occurrences into UTC instants before delivery, and make a queue worker consume those immutable occurrences with an idempotency key and an audit trail.

Don't ask a cron process to remember what was promised to a user. Cron should wake a planner; the planner should materialize promises; the worker should deliver them. This separation works for a Node.js service even though the focused example below is in Go, and it keeps US and EU daylight-saving changes out of the delivery path.

The hard constraint is semantic: “Monday at 09:00” is a recurring civil-time instruction, while `2026-08-10T07:00:00Z` is one instant. An offset such as `UTC-04:00` isn't a timezone rule, because it cannot describe future clock changes. Store the local hour, recurrence, and IANA zone as the source rule; store each resolved UTC instant as a derived occurrence. If the user's intended zone changes, version the rule and regenerate only future, unsent occurrences.

## What should timezone-aware Node.js reminders, cron, and queue workers each own?

The Node.js application should own the user-facing rule and its lifecycle. A periodic cron trigger should own only a lease-protected request to plan a bounded future window. A planner should own timezone conversion and occurrence creation. A queue worker should own delivery state, retries, and evidence. That division matters because timezone decisions can then be tested as deterministic input and output, while queue redelivery becomes a state-machine problem rather than another calendar calculation.

Use a stable model with three records. `reminder_rule` contains the user's timezone, local clock, daily or weekly cadence, selected weekdays, status, and revision. `reminder_occurrence` contains the rule revision, intended local date and time, resolved UTC instant, resolution policy, and a unique occurrence key. `delivery_attempt` contains the occurrence key, attempt identifier, timestamps, and outcome. The occurrence is the audit boundary: it records what the system intended before any channel call happened.

Exactly once is the mindset, not a claim about the transport.

Give each occurrence a deterministic key such as `(rule_id, rule_revision, intended_local_date, intended_local_time)`, protected by a database unique constraint. Planning the same window twice then converges on the same rows. Enqueueing should use a transactional outbox, or an equivalent atomic database transition, so a committed occurrence cannot be silently separated from its enqueue intent. On consumption, claim the occurrence conditionally; if it is already terminal, acknowledge the duplicate without delivering again.

The clock and recurrence rule belong at the edge of the system. The worker receives `occurrence_id`, `due_at`, and a delivery payload reference. It should reject work that is not yet due, extend its lease when processing approaches the queue's visibility deadline, and record a channel acknowledgment before marking the occurrence delivered. AWS documents that a received SQS message remains temporarily invisible and may become available again if it is not deleted before the visibility timeout; that is why a visibility lease cannot substitute for application-level idempotency.

## DST is a policy decision, not a date-library setting

Two kinds of local time require an explicit product policy. During a forward clock change, a requested wall time can be nonexistent. During a backward change, a wall time can be ambiguous because it occurs twice. A library can detect or resolve these cases, but it cannot decide what the user meant.

No guesswork.

For a nonexistent daily reminder, plausible policies include moving to the first valid instant after the gap or skipping that local date. For an ambiguous reminder, plausible policies include choosing the earlier occurrence or the later one. There is no universal answer. A medication prompt, a payout-approval reminder, and a weekly digest can reasonably choose different behavior. Record the selected policy on the rule and the applied result on every occurrence, because otherwise an auditor can see the UTC timestamp but cannot reconstruct why it was selected.

US and EU users also cannot be handled by a shared “DST mode.” Their zones follow their own transition histories, and a timezone database update may change future mappings. Your mileage may vary by jurisdiction, so the operational rule should be simple: resolve from the named zone installed in the planning runtime, retain the timezone-data version when practical, and treat a timezone-data update as a reviewed change that can re-plan future occurrences without rewriting delivered history.

Test transitions with fixtures, not with the current wall clock. For every supported policy, include an ordinary date, a gap, an overlap, a weekly recurrence that crosses each case, and a rule revision made after planning but before delivery. Also test distinct US and EU zones in the same planning batch. I'm not sure any finite fixture set can anticipate legislative changes; pinning the timezone data in CI and reviewing updates is what makes that uncertainty visible.

The resolver contract can remain small even when the implementation uses a mature timezone library in the Node.js planner. The Go below defines the records and invariants without pretending that a fixed UTC offset is enough:

```go
package reminders

import (
	"context"
	"errors"
	"time"
)

type GapPolicy string
type OverlapPolicy string

const (
	MoveForward GapPolicy = "move_forward"
	SkipDate    GapPolicy = "skip_date"
	Earlier     OverlapPolicy = "earlier"
	Later       OverlapPolicy = "later"
)

type LocalRule struct {
	RuleID        string
	Revision      int64
	Timezone      string // IANA name, for example America/New_York.
	Hour          int
	Minute        int
	GapPolicy     GapPolicy
	OverlapPolicy OverlapPolicy
}

type Occurrence struct {
	Key               string
	IntendedLocalDate string
	IntendedLocalTime string
	DueAt             time.Time
	Resolution        string
}

type Resolver interface {
	Resolve(rule LocalRule, localDate string) (Occurrence, error)
}

type Store interface {
	InsertOccurrence(ctx context.Context, occurrence Occurrence) (inserted bool, err error)
	ClaimForDelivery(ctx context.Context, key string, leaseUntil time.Time) (claimed bool, err error)
	MarkDelivered(ctx context.Context, key, channelReceipt string, deliveredAt time.Time) error
}

var ErrMissingReceipt = errors.New("channel receipt is required")
```

Notice what the interface refuses to do: it does not accept a raw numeric offset, and it does not let the delivery worker resolve a local date. Those omissions prevent two entire failure classes. In a Node.js implementation, the same contract should sit behind tests that round-trip the requested calendar fields in the named zone and require an explicit result for gaps and overlaps.

## How should a queue worker handle daily and weekly local-time DST delivery?

It should handle neither recurrence nor DST. It should claim one already-resolved occurrence, perform the channel operation, persist the acknowledgment, and tolerate receiving the same queue message again. Daily versus weekly changes how the planner selects local dates; it does not change the delivery protocol.

```go
package reminders

import (
	"context"
	"time"
)

type Message struct {
	OccurrenceKey string
	UserID        string
	PayloadRef    string
}

type Channel interface {
	Send(ctx context.Context, idempotencyKey, userID, payloadRef string) (receipt string, err error)
}

type Worker struct {
	Store   Store
	Channel Channel
	Now     func() time.Time
}

func (w Worker) Handle(ctx context.Context, message Message) error {
	claimed, err := w.Store.ClaimForDelivery(
		ctx,
		message.OccurrenceKey,
		w.Now().Add(2*time.Minute),
	)
	if err != nil || !claimed {
		return err
	}

	receipt, err := w.Channel.Send(
		ctx,
		message.OccurrenceKey,
		message.UserID,
		message.PayloadRef,
	)
	if err != nil {
		return err
	}
	if receipt == "" {
		return ErrMissingReceipt
	}

	return w.Store.MarkDelivered(ctx, message.OccurrenceKey, receipt, w.Now())
}
```

There is a sharp limit here: a local claim prevents concurrent sends inside your system, but it cannot erase the crash interval between a channel accepting a message and the database recording its receipt. Pass the occurrence key downstream as an idempotency key when the channel supports that contract. When it doesn't, honest at-least-once delivery plus reconciliation is safer than advertising exactly-once behavior that the system cannot prove.

Retries need classification. A transient transport failure may be retried with bounded backoff while the occurrence remains eligible; a permanent recipient or policy rejection should become a terminal, reviewable outcome. Never encode retry state only in queue receive counts. Persist attempts against the occurrence so operators can answer which reminder was due, which payload revision was used, how many attempts occurred, and what acknowledgment was retained. Consider a weekly 09:00 reminder whose rule is edited at 08:57 after its occurrence has already entered the outbox: the occurrence must retain the rule revision and payload reference selected at planning time, the edit must create a new revision rather than mutating that evidence, and cancellation policy must decide whether the claimed occurrence remains valid. At 09:01, a redelivered queue message must point to the same occurrence key; it must not re-read the current rule and silently transform an old promise into a new one. If the channel accepted the first attempt but the receipt write did not complete, reconciliation should expose an indeterminate state for review rather than manufacture certainty. This one scenario exercises rule versioning, cancellation, queue duplication, downstream idempotency, and audit retention, which is why a generic `sendReminder(userID)` job is too weak a contract even when its calendar calculation is correct. Observability should follow those same transitions: measure planning lag, due occurrences without an outbox event, queue age, claim contention, delivery latency, duplicate claims suppressed, terminal failures, and the difference between due and delivered counts per bounded time window. Reconciliation is an independent query that compares promises with outcomes and can page an operator when the sets diverge.

## Choosing the scheduler and defining its limits

The scheduling mechanism is now a smaller decision because it does not carry user timezone semantics. A host cron, a container-platform timer, or a managed schedule can all wake the planner if the trigger is authenticated, monitored, and safe to repeat. GitHub Actions documents scheduled workflow execution in UTC and warns that scheduled runs can be delayed during high load; that makes it reasonable for non-urgent maintenance or a backfill trigger, but it is a poor boundary for a customer promise that must be evaluated at a precise local minute.

| Constraint | Suitable design | The catch |
| --- | --- | --- |
| Small installation, reminders tolerate planning delay | One recurring planner plus a database outbox | The database is both the audit source and a scaling boundary |
| Large fan-out with retryable delivery | Partitioned planners, occurrence rows, queue consumers | Partition ownership and reconciliation require operational discipline |
| Very high-frequency signals | Windowed or streaming computation with a delivery ledger | A row for every future occurrence may be unsuitable |
| Best-effort internal housekeeping | A UTC scheduled workflow | Delayed execution and weak per-user auditability may be acceptable only here |

The occurrence-ledger design is not suitable when schedules change many times per second or when retaining one row per planned event exceeds the storage and write budget. In that regime, compute shorter windows or stream the schedule, but keep a durable delivery ledger and deterministic keys. Conversely, stick with a plain UTC timer when the task is truly system-wide, has no user-local meaning, and can tolerate delay; introducing timezone planning there adds state without buying correctness.

Cost analysis should include occurrence writes, outbox writes, queue operations, attempt retention, reconciliation scans, and on-call complexity. Don't optimize away the evidence first. Retention may also be constrained by privacy policy and sector-specific recordkeeping rules, so keep delivery payloads separate from minimal audit metadata, define deletion schedules, restrict access, and have compliance counsel validate local-time contact windows for every channel and jurisdiction. The scheduler can enforce a configured window; it cannot determine the law.

Roll out in compact stages. First, run the planner in shadow mode and compare its proposed instants with the current path without enqueueing. Then enable one timezone cohort, reconcile every due occurrence against its delivery result, and expand by region. During migration, assign a single authority per rule revision so old and new schedulers cannot both send. Keep a kill switch that stops new claims without deleting the occurrence ledger; evidence should survive an operational pause.

This architecture is intentionally plain. The payoff is that a support or audit query can move from rule, to local-time decision, to UTC occurrence, to queue attempt, to delivery receipt without reconstructing history from logs. That is the standard a recurring user reminder should meet.

## Further reading

- [AWS SQS visibility timeout documentation](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-visibility-timeout.html)
- [GitHub Actions workflow triggers documentation](https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows)
