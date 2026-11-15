# Gaming OTP Reset Backend Flow in 2026 — Auditing Failed SMS Delivery Status

A gaming account's password-reset credential must expire even when its SMS delivery status arrives late. That constraint determines the backend flow: the authority to reset a password belongs to the credential store, while delivery notifications supply evidence about a communication attempt. SMS 2FA login needs the same separation between communication evidence and authentication authority.

TL;DR: Persist a reset intent and its dispatch work together, assign each send attempt a stable identifier, and reconcile delivery notifications or status polls against that attempt. Keep credential expiry and single use independent of delivery status. A failed OTP send can justify another communication attempt under an explicit policy; it cannot justify silently extending the credential's lifetime. The evidence should explain each decision without retaining the OTP itself.

This note uses a gaming password reset as the concrete case. The same separation helps with SMS 2FA login, although a successful password reset must not be treated as evidence that a separate second authentication factor was satisfied. The examples below are proposed design choices, including a hypothetical 5-minute expiry, rather than claims about a messaging service or a universal security requirement.

## Start with the expiry constraint

OWASP's Forgot Password Cheat Sheet calls for reset tokens that are securely generated, stored securely, single use, and time limited; it also recommends consistent responses for existing and nonexistent accounts and controls against excessive requests. Those requirements make delivery status a subordinate input: a delivery receipt cannot establish that the claimant controls the account, and a delayed message cannot change the time at which its credential expires.

Expiry wins here.

For the illustrative game account `player-1842`, define reset generation `g17`, logical notification `reset-1842-g17`, and an expiry recorded when the credential is issued. In this hypothetical policy, the credential lasts 5 minutes, or 300 seconds. A resend of that credential retains that expiry. If policy permits issuing a replacement, create another generation and atomically invalidate the previous credential, so the audit trail can distinguish a repeated dispatch from a new authorization opportunity. The trade-off is reduced user convenience in exchange for a simpler answer to the support question: which credential could have changed this account's password at this instant?

Store a verifier suitable for the credential's entropy and threat model. For a short numeric OTP, a bare fast hash does not make offline guessing expensive; use a design that protects the verifier against that risk, limit verification attempts, and keep secrets out of logs. The choice of storage protection needs security review. The receipt ledger should carry opaque identifiers, restricted destination references, timestamps, and reasons, rather than the reset token or message body.

## What does a delivery receipt actually prove?

Treat the sender's acceptance, a later delivery observation, and successful credential redemption as different events. The meaning of a particular delivery status depends on the transport contract; the supplied sources do not establish any SMS service's status vocabulary, terminal states, callback authentication mechanism, or polling limits. Verify those details before implementing an adapter, and retain the original status alongside its documented interpretation.

A reset is still pending after a message is reported delivered.

That distinction is central to compliance evidence. An audit entry should identify what was observed, when the system received it, which attempt it concerned, and which policy decision followed. Where a source provides its own event time, record it separately from receipt time. Neither clock alone establishes a total ordering across independent systems, so an explicit reconciliation rule is more defensible than whichever callback happened to update a row last.

Consider the concrete pitfall of an old failed attempt arriving after a replacement attempt has been accepted. Updating a single account-level `delivery_failed` flag would misdescribe the current reset. Attach the failure to its original attempt, then derive the user-visible state from the active generation and its relevant attempts. Preserve contradictory observations for investigation instead of deleting the inconvenient event.

## Derive the send path from the evidence requirements

Use one local transaction to create the credential record, notification intent, and dispatch work item. A worker can then claim that work, record an attempt, and contact the transport. This proposed outbox arrangement makes the decision to send recoverable after a process restart; it does not make the external send part of the local database transaction.

The difficult boundary is an ambiguous response. If the connection ends without a conclusive acknowledgement, the sender may already have accepted the message. Record the outcome as unknown. Where the documented transport contract supports idempotent requests, preserve the same request key while resolving that attempt; otherwise, consult its supported status lookup before deciding whether the risk of a duplicate message is acceptable. Creating a fresh attempt immediately can convert uncertainty into duplicate sends.

Keep an exactly-once mindset for local decisions, with explicit limits at the external boundary. A unique logical intent prevents duplicate local scheduling, and an atomic redemption prevents two successful uses of one credential. Neither proves exactly-once SMS delivery.

The following Go example specifies a storage boundary for callback ingestion. Its atomic operation is a contract that the database implementation must satisfy, including concurrent duplicate notifications; it is not a complete persistence or transport implementation. An authenticated transport adapter must first verify the notification and resolve its attempt identifier from trusted local correlation data.

```go
package reset

import (
	"context"
	"errors"
	"time"
)

type Receipt struct {
	Source     string
	EventID    string
	AttemptID  string
	RawStatus  string
	ReceivedAt time.Time
}

type ReceiptLedger interface {
	// Atomically deduplicate by (Source, EventID), append the observation,
	// and reconcile its attempt. Reject unknown attempt identifiers.
	// Never change credential expiry or mark a credential redeemed.
	AppendAndReconcile(context.Context, Receipt) (inserted bool, err error)
}

func RecordReceipt(
	ctx context.Context,
	ledger ReceiptLedger,
	r Receipt,
	now time.Time,
) (bool, error) {
	if r.Source == "" || r.EventID == "" ||
		r.AttemptID == "" || r.RawStatus == "" {
		return false, errors.New("incomplete delivery observation")
	}
	r.ReceivedAt = now.UTC()
	return ledger.AppendAndReconcile(ctx, r)
}
```

This example assumes the adapter can obtain a stable source event identifier. If the transport does not supply one, the adapter needs a documented deduplication strategy; concatenating a status and timestamp without examining their uniqueness is insufficient. Polling observations also need their own identity scheme, since repeatedly observing the same status need not mean that the upstream system emitted multiple events.

## How should a backend flow poll SMS delivery status before retrying?

Once the ledger owns the decision history, choosing callbacks or polling becomes an operational question. Both must enter the same reconciliation path, so two sources cannot independently authorize a resend.

| Method | Evidence to retain | Engineering trade-off |
| --- | --- | --- |
| Delivery notifications | Authenticated source, event identity, attempt mapping, raw status | Requires an ingress path, replay handling, and monitoring for missing observations. |
| Status polling | Attempt mapping, lookup time, returned status, lookup outcome | Requires scheduled work and an explicit query budget within documented limits. |
| Notifications with reconciliation polling | Both observation types and the rule used to resolve them | Adds recovery opportunities and additional implementation and operational cost. |

For this reset design, schedule bounded reconciliation for unresolved attempts, with timing chosen from the transport's documented limits and the credential's remaining lifetime. A lookup failure is evidence that the lookup failed. It is not evidence that the SMS failed. Stop scheduling user-facing resend work after expiry; a separately bounded evidence-reconciliation task may continue if the retention policy and incident workflow require it.

The trade-off is labor as well as traffic: an authenticated callback receiver adds deployment and security-review work, while polling adds scheduler state, request volume, and rate-limit handling. Estimate those costs from expected reset volume and the selected contract. Price alone cannot resolve whether the evidence is sufficient.

If email is an approved recovery channel, give it a separate attempt and preserve the same credential rules. Sender requirements become an additional constraint: Yahoo's published guidance addresses email authentication and sender practices, with requirements that vary by sender category. That guidance does not establish SMS delivery semantics or prove that a recipient redeemed a reset credential.

## Roll out with evidence that can survive review

First test duplicate receipts, receipts arriving out of order, an ambiguous send response, a worker restart after acceptance, and redemption racing with expiry. For the last case, the storage operation must check validity and consume the credential atomically. A test should demonstrate one successful redemption at most, without assuming that successful delivery permits redemption after expiry.

Then deploy reconciliation in observation mode, comparing derived states with the existing send records before it controls retries. Monitor unresolved attempts, receipt lag, duplicate observations, expired dispatch work, and reconciliation failures. Keep authentication responses consistent, and expose only suitably scoped delivery information to an authenticated recovery session. Do not turn a delivery-status endpoint into an account-discovery mechanism.

Finally, enable resend decisions gradually under a versioned policy, with a rollback control that stops new dispatch decisions while allowing evidence ingestion to continue. Review retention, access restrictions, and deletion requirements with the responsible compliance team: an append-oriented audit trail is an engineering property, not a certification or a reason to retain personal data indefinitely. The acceptance criterion is a reconstructable reset decision, bounded credential validity, and a recorded outcome for every attempted communication.

## Sources

References:

- OWASP, Forgot Password Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html
- Yahoo, Sender Best Practices and Requirements: https://senders.yahooinc.com/best-practices/
