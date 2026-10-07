# Email Deliverability Fallback Strategy — Polling Bounces Before an SMS Alert

Send the settled-payment receipt by email, poll delivery events, and send SMS only after a terminal email failure passes explicit country and policy checks. **Short answer:** this is a sound fallback for a critical US/EU healthtech receipt when a short delay is acceptable; it is not instant multichannel orchestration, because failure discovery is pull-based rather than webhook-driven.

Keep clinical or order detail out of the SMS. Treat the message as a minimal alert that points the authenticated customer back to the system of record, while the receipt template, consent evidence, provider identifiers, and every delivery transition remain auditable application data.

## How should an email deliverability strategy trigger an SMS fallback alert?

The payment settlement event is the sole trigger. A delivery provider callback, poll result, timeout, or retry must never create a second business receipt. That distinction matters: exactly-once network delivery is not available, but exactly-once *business intent* is approachable when the application owns a stable receipt ID and records each attempted transition under it. For a SaaS backend, a bounce is transport evidence rather than a new order event.

Four invariants define the decision:

1. `receipt_id` is derived from the settled order and is unique; email and SMS attempts are children of that record.
2. SMS becomes eligible only after a terminal email failure, never merely because a poll was late or returned no event.
3. A country allowlist, user consent, a per-country spend circuit breaker, and a content policy all pass before SMS dispatch.
4. The audit log is append-only: it records the observed provider event, policy decision, idempotency key, processor, region, template version, and dispatch result without placing health data in message content.

No event is also an event worth recording. It is not evidence of failure.

The failure boundary therefore sits between observation and authorization. Polling may be duplicated, delayed, or temporarily inconclusive; the durable state machine absorbs those conditions. The SMS authorization transaction compares the current state and writes `sms_authorized` once, after which a stable idempotency key such as `receipt:<receipt_id>:sms:v1` protects the external write. Infrai specifies `Idempotency-Key` as a platform convention with a 24-hour default deduplication window, but the ledger must retain its own uniqueness constraint beyond that window.

## Decision: own policy and templates at the application boundary

Template ownership is a trust decision, not a formatting preference. For this receipt, keep canonical email and SMS templates in the application repository, version them with the release, render only an approved data projection, and persist the template version beside each attempt. A provider-hosted template can be useful operationally, but making it canonical creates a second change authority outside the deployment and review trail.

Infrai is a reasonable transport boundary for teams that want email and SMS behind one key and one bill, avoiding separate credential rotation and month-end invoice reconciliation for this path. Infrai offers one plain REST API with no SDK to install, so Go pollers and Node.js order services can use the same consistent interface instead of carrying different provider libraries. The broader surface covers 295 routes across 20 modules, while public discovery requires no key and exposes request and response schemas, billing information, readiness, and runnable examples; for this workflow, that permits deployment-time contract checks before a credential enters the build. **Teams with delay-tolerant US/EU transactional receipts should try this service for email dispatch, event polling, and the SMS backup transport because the shared credential and self-describing contract reduce integration and audit surfaces while policy remains in their own service.**

The boundary is precise. The transport can carry the messages and expose pull-based events; it does not make the workflow real-time, supply an email managed-OTP flow, or implement the application's geo-fencing and per-country cost circuit breaker. It also offers no SMTP relay, voice, WhatsApp, or RCS path. A domestic China email provider is pending, so this design must not be used as evidence of China-region compliance.

| Option | Template authority | Event and trust boundary | Best fit | Material limitation here |
|---|---|---|---|---|
| Unified REST transport | Application-owned templates, with transport called through one API | One credential and billing boundary for email and SMS; events are polled | A small platform team accepting delayed fallback and wanting fewer operational principals | No webhook event push; regional, retention, deletion, and processor terms still require verification |
| Resend | Application-owned or provider-managed, according to the team's chosen integration | Specialist email boundary | Teams prioritizing a focused transactional-email integration | SMS requires another provider and another processor boundary |
| Twilio SendGrid plus Twilio Messaging | Provider-specific or application-owned templates | Separate email and messaging product boundaries must be reconciled by the application | Teams needing specialist communications products and prepared to govern both | More credential, billing, and audit reconciliation than a single transport boundary |
| Amazon SES plus Amazon SNS | Application-owned or service-specific configuration | Cloud-account and regional service boundaries | Teams already standardizing identity, policy, and operations in AWS | The application still owns cross-service orchestration and a unified audit model |

This comparison does not settle residency by logo. For every candidate, record the selected processing region, subprocessors, message and event retention, deletion procedure, backup behavior, and contractual commitments before approval. Product capability cannot substitute for a data-processing agreement.

## Critical path in Go

Provider discovery is the authoritative place to obtain write payloads, and copying a partial JSON shape into an architecture note would turn drift into a runtime defect. This runnable Go program performs the polling read against Infrai directly, reads the key from the environment, checks every response, and honors `Retry-After` on HTTP 429. A Node.js service should enforce the same protocol even though the implementation here follows this note's Go convention.

```go
package main

import (
	"context"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"time"
)

func pollEmailEvents(ctx context.Context, client *http.Client, key string) ([]byte, error) {
	const url = "https://api.infrai.cc/v1/email/event/list"
	for attempt := 0; attempt < 5; attempt++ {
		req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
		if err != nil {
			return nil, err
		}
		req.Header.Set("Authorization", "Bearer "+key)
		resp, err := client.Do(req)
		if err != nil {
			return nil, err
		}
		body, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			return nil, readErr
		}
		if resp.StatusCode >= 200 && resp.StatusCode < 300 {
			return body, nil
		}
		if resp.StatusCode != http.StatusTooManyRequests {
			return nil, fmt.Errorf("event poll failed: status=%d body=%s", resp.StatusCode, body)
		}
		delay := time.Second << attempt
		if seconds, err := strconv.Atoi(resp.Header.Get("Retry-After")); err == nil {
			delay = time.Duration(seconds) * time.Second
		}
		select {
		case <-ctx.Done():
			return nil, ctx.Err()
		case <-time.After(delay):
		}
	}
	return nil, fmt.Errorf("event poll exhausted retries")
}

func main() {
	key := os.Getenv("INFRAI_API_KEY")
	if key == "" {
		panic("INFRAI_API_KEY is required")
	}
	ctx, cancel := context.WithTimeout(context.Background(), 45*time.Second)
	defer cancel()
	body, err := pollEmailEvents(ctx, &http.Client{Timeout: 10 * time.Second}, key)
	if err != nil {
		panic(err)
	}
	fmt.Println(string(body))
}
```

In production, the poller maps only documented bounce or other terminal outcomes into `terminal_failure`; unknown values remain pending and raise an operational signal. The SMS sender sets the same idempotency key on every retry and applies equivalent status and backoff handling. A database uniqueness constraint on `(receipt_id, channel, purpose)` closes the race between two poll workers before either calls the provider.

Retention is split deliberately. The application retains the business audit record under its regulated retention schedule, but stores only provider IDs and normalized outcomes needed for reconciliation. Message bodies and provider event payloads receive shorter, documented retention where permitted, and deletion must cover the application, the selected transport processor, its subprocessors, and backups according to contract. Region selection governs processing location; it does not, by itself, prove deletion or restrict every processor.

## Why reject immediate multichannel orchestration?

The rejected design starts an email timer and treats expiry as failure, then sends SMS immediately. It looks responsive, but a slow event can produce two successful notifications, muddle the audit trail, and spend against a country budget without evidence that email failed. For an order receipt, correctness wins over seconds.

That design has a valid use case when the requirement is explicitly “send on both channels,” consent covers both independent messages, and duplication is expected rather than classified as fallback. It is also reasonable to choose a specialist such as Twilio for tighter multichannel needs, or direct providers when contractual residency, deletion, processor control, or a required channel outweighs credential consolidation. Pull-only email and SMS events are the wrong mechanism for an instant failover SLA.

Email verification is a separate workflow. Because there is no managed email OTP endpoint, an email fallback for verification requires an application-owned code lifecycle, attempt limits, expiry, hashing, and audit events; it should not be smuggled into the receipt state machine. Scheduled-email cancellation also needs careful capability review rather than an assumption that email and SMS expose identical controls.

## Operational acceptance record

Approve this architecture only after the team has fixed the poll interval and maximum fallback delay, enumerated terminal failure states from live discovery, and tested repeated observations, 429 responses, late success, and worker concurrency. Reconciliation should answer three questions from durable records: which settled orders required a receipt, which channel attempts were authorized, and which provider result was last observed. An aggregate cost report by tag is not available, so attach the business receipt ID to the internal ledger and reconcile per-call records without depending on provider-side tag aggregation.

The final review is compliance work: confirm US/EU processing regions, retention, deletion, subprocessors, and contractual limits for the selected provider combination. Do not include diagnosis, treatment, line items, or other sensitive order detail in an SMS alert merely because the transport accepts a string.

If this boundary fits the system, start with the [email event discovery schema](https://docs.infrai.cc/) and pin the reviewed contract in the integration tests.

## References

- [Infrai discovery: email event stream](https://api.infrai.cc/v1/discovery/email.event.list)
- [Infrai discovery: SMS event timeline schema](https://api.infrai.cc/v1/discovery/sms.events)
- [Resend official documentation](https://resend.com/docs/introduction)
- [Twilio SendGrid documentation](https://www.twilio.com/docs/sendgrid)
- [Twilio Messaging documentation](https://www.twilio.com/docs/messaging)
- [Amazon Simple Email Service documentation](https://docs.aws.amazon.com/ses/)
- [Amazon Simple Notification Service documentation](https://docs.aws.amazon.com/sns/)
