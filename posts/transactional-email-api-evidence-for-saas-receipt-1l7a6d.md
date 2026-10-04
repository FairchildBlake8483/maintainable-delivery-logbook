# Transactional Email API Evidence for SaaS Receipts — Regional Delivery After Settlement

**TL;DR:** choose a transactional email API by asking whether an operator can prove what happened after an ambiguous send, not by comparing the cheapest headline tier. For a logistics platform that emits an order receipt only after payment settles, the sound design fixes one logical receipt identity, records an append-only evidence trail, and tests every candidate transport—including Resend, Postmark, SendGrid, and MailerSend—against the same failure script in both required regions. The transport can accept a message. It cannot own the business fact that the receipt was due.

That distinction resolves the main trade-off. A slightly easier integration has little value if a timeout leaves the team unable to decide whether retrying will duplicate a financial communication, while a feature-rich service still fails the system boundary if its events cannot be reconciled to the settlement that authorized the receipt. Price belongs near the end of this decision because the unit being purchased is not merely an API request; it is a recoverable, auditable outcome.

## How should a SaaS transactional email API prove receipt outcomes?

Start with the questions an on-call engineer or auditor will ask: Did settlement `stl_8142` authorize a receipt? Which immutable content version represented order `ord_2719`? Did the transport accept an attempt, did a later event identify a permanent failure, and who approved any replay? These are separate propositions. A queue acknowledgement proves none of the remote ones, and an API acceptance does not prove inbox placement or human reading.

The evidence model therefore needs three identities. The logical receipt key names the single business obligation, an attempt ID names each network operation, and a transport message ID links later provider evidence when one is returned. Keep them separate. If a client deadline expires after the remote service accepts the message but before the response arrives, the attempt is `unknown`, not `failed`; automatically converting that ambiguity into a retry sacrifices duplicate control for apparent availability. This is an exactly-once mindset applied honestly: the external effect cannot be made atomic with the settlement transaction, but authorization and evidence can be made idempotent locally. A unique constraint on `(settlement_id, message_kind)` prevents two logical receipts, while an append-only transition log makes reclassification visible. The current state is a projection, not the audit record.

Unknown is evidence.

DKIM covers a different claim. RFC 6376 defines a domain-level cryptographic signature that permits a verifier to associate a message with a signing domain; it does not establish delivery, content retention, or readership. Treat selector inventory, key rotation, and verification after DNS changes as release controls, then retain the verification result alongside the deployment record.

## Make ambiguity executable

A provider comparison becomes useful only after uncertainty is represented in code. The following Go example deliberately refuses to send when an earlier attempt has an unresolved outcome. It does not assume a vendor-specific idempotency feature, endpoint, or event vocabulary.

```go
package receipt

import (
	"context"
	"errors"
	"fmt"
	"time"
)

type State string

const (
	Pending  State = "pending"
	Unknown  State = "unknown"
	Accepted State = "accepted"
	Rejected State = "rejected"
)

type Attempt struct {
	LogicalKey string
	AttemptID  string
	State      State
	StartedAt  time.Time
}

type Store interface {
	Load(context.Context, string) (Attempt, error)
	Start(context.Context, string) (Attempt, error)
	Append(context.Context, string, State, string) error
}

type Transport interface {
	Send(context.Context, string, []byte) (string, error)
}

var ErrReconcileFirst = errors.New("receipt outcome is unknown; reconcile before replay")

func Dispatch(ctx context.Context, db Store, mail Transport, key string, body []byte) error {
	current, err := db.Load(ctx, key)
	if err == nil && current.State == Unknown {
		return ErrReconcileFirst
	}
	if err == nil && current.State == Accepted {
		return nil
	}

	attempt, err := db.Start(ctx, key)
	if err != nil {
		return err
	}
	messageID, err := mail.Send(ctx, attempt.AttemptID, body)
	if err != nil {
		if auditErr := db.Append(ctx, attempt.AttemptID, Unknown, err.Error()); auditErr != nil {
			return fmt.Errorf("send: %v; audit: %w", err, auditErr)
		}
		return err
	}
	return db.Append(ctx, attempt.AttemptID, Accepted, messageID)
}
```

The short branch for `Unknown` is the important part. The implementation should also distinguish a definitive local validation rejection from an uncertain network result, verify incoming event authenticity according to the selected transport's current documentation, deduplicate events, and preserve raw events before deriving state. Sensitive receipt content does not belong in error strings or general-purpose logs; retention, access, and regional processing limits must come from the organization's data map, legal obligations, and contracts.

Use fixed fault cases rather than optimistic mocks. Let the remote call run while imposing a 30-second client deadline as a test parameter, interrupt the worker after acceptance but before the local append, deliver the same event twice, reverse two event arrival times, and replay the same logical key. The pass condition is deterministic: no test may create a second logical receipt, and no unknown attempt may be replayed until evidence or an authorized operator resolves it.

One detail deserves disproportionate attention. Store the template version and the minimal receipt data needed to reproduce the artifact at authorization time; rendering whatever template happens to be current when a delayed worker runs can produce an email that no longer matches the recorded order terms.

## Compare evidence paths before developer convenience

Resend, Postmark, SendGrid, and MailerSend should enter the exercise as unnamed rows before their results are collected. The cited public documentation establishes that Resend exposes an email API, but it does not substantiate a complete four-way verdict, and product behavior or contractual terms can change. A durable engineering note should define the questions and require dated primary evidence rather than manufacture a ranking from uneven documentation.

| Gate | Experiment or document | Required evidence |
| --- | --- | --- |
| Ambiguous acceptance | Complete a request while the client observes only its deadline | A defensible path from attempt ID to accepted, rejected, or still unknown |
| Duplicate resistance | Submit the same logical key through concurrent workers | One business obligation and an auditable record of every attempt |
| Event integrity | Send duplicate and out-of-order callbacks | Verified source, raw event retention, deduplication, and monotonic projection rules |
| Domain authentication | Configure and rotate a DKIM selector | Successful verification and a recorded rollback procedure |
| Regional governance | Review the current contract and data-flow documentation | Documented locations, retention, subprocessors, access, and transfer basis as applicable |
| Operations | Reconcile a deliberately stranded attempt | An operator can resolve it without editing history or guessing from logs |

Run the matrix with separately authenticated test domains and controlled recipient accounts for the US and EU cohorts the service actually supports. Record the date, configuration, request correlation data, observed events, and unresolved questions. Do not equate geography in a dashboard with a compliance conclusion: message bodies, suppression records, telemetry, backups, and support access may have different processing paths, and only current contractual and technical evidence can settle the applicable boundary. Developer experience follows the hard gates. Compare the effort to maintain a narrow Go adapter, rotate credentials, verify events, isolate tests, export evidence, and diagnose throttling without exposing customer data. Then model cost against the actual traffic distribution and operational burden. No lowest-price claim remains stable enough to substitute for this work.

## Roll out the evidence contract in four moves

First, define the logical key, allowed states, transition rules, template version, and retention policy without changing the active sender. Second, shadow-write evidence for the existing path and reconcile it against settled orders; any gap is a schema or instrumentation problem, not permission to infer delivery. Third, enable dispatch for a small deterministic cohort, with unknown outcomes held for review and expansion tied to written thresholds for unresolved age, duplicate logical receipts, and missing evidence. Fourth, exercise another adapter with isolated recipients before treating transport replacement as an operational capability.

Keep the rollback boundary narrow: disable new dispatch while retaining the ledger and raw evidence. Do not delete or rewrite an inconvenient transition.

**The selection is complete when the team can explain every tested receipt from settlement authorization through its latest known transport outcome, under the required US and EU controls, without relying on a provider dashboard as the system of record.** That criterion does not crown a universal winner. It produces a decision that can be reviewed, repeated, and reversed when evidence changes.

## Sources

- https://datatracker.ietf.org/doc/html/rfc6376
- https://resend.com/docs/introduction
