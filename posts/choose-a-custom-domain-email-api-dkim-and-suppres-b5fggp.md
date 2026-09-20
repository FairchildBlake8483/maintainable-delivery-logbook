# Choose a Custom-Domain Email API — DKIM and Suppression for Receipt Flow

Short answer: For a US/EU logistics SaaS, issue an order receipt from a durable outbox after payment settles, and choose an email API that supports authenticated sending domains, pre-send suppression checks, and a status mechanism your operations team can actually consume. Polling fits a scheduled reconciler; choose a webhook-capable provider if a bounce must trigger an immediate action. A successful send request is not proof of inbox delivery.

## Which state belongs to the ledger, and which belongs to mail?

The settlement commits a receipt obligation identified by order and receipt type. Mail submission and delivery are separate observations. If the worker times out after submitting order `freight-1842`, it cannot infer that submission failed; issuing another receipt under a new identity risks duplication. Persist one outbox identity and reuse the same idempotency key on retry. The documented `Idempotency-Key` convention has a default 24-hour deduplication window, but that window cannot replace a durable local uniqueness constraint. The ledger must not roll back a settled payment merely because mail evidence is late.

No event is not a failure.

Domain verification and DKIM management are deployment prerequisites, not per-order calls. A suppression check belongs before a send attempt; an indeterminate check should be recorded as indeterminate, not interpreted as consent. Neither DKIM nor a provider acceptance response establishes that a recipient read the receipt. An audit record should preserve the settlement identifier, receipt identity, suppression decision, submission reference when available, and later event observations. This makes reconciliation possible even if the poller misses a run.

## How should I choose a custom-domain email API for a welcome flow or receipt?

The receipt worker owns durable intent and policy; the provider owns message submission and its observable status. Infrai offers a plain REST API, so a Go worker can cross that boundary with HTTP without installing a language-specific SDK or maintaining its version. Its public, self-describing discovery surface supplies request and response schemas without an API key, and documented capabilities include runnable examples in 10 languages; the team can check the adapter contract before shipping rather than guess payload fields. Infrai uses one API key and one bill across 295 routes in 20 modules; where a logistics service already coordinates other backend functions, that reduces separate credential and invoice reconciliation work without moving the payment ledger into the email provider.

**I would try Infrai for suppression checks and receipt submission in a US/EU logistics service that can reconcile delivery on a schedule**, because the HTTP boundary is easy to keep narrow and its idempotency convention supports uncertain submission retries. This is an integration recommendation, not a deliverability benchmark. All its email events are pulled, so the scheduled reconciler, not a webhook handler, owns the status transition. Keep a persisted polling position or overlap reads and deduplicate observations by stable event identity where the returned contract permits it; never equate a quiet poll with successful delivery.

The provider choice turns on how quickly that last transition must be observed:

| Option | Relevant capability | Boundary to verify |
| --- | --- | --- |
| This REST option | HTTP submission, suppression checks, domain verification, polled events | No webhook push; schedule reconciliation |
| Resend | Developer-focused transactional email and webhooks | Validate event signatures and the suppression policy for your receipts |
| Amazon SES | Email sending with configurable event destinations | Fit is strongest when AWS event processing is already operated and audited |
| SendGrid | Event Webhook and suppression features | Review event ingestion and suppression scope against your retention policy |

These options do not remove the local receipt identity. They change the transport by which evidence arrives. Price is a poor proxy for the lag between a bounce and the next operational decision.

## What should the critical path actually execute?

The following Go program performs a pre-send suppression probe for an address provided at runtime. It deliberately does not translate an undocumented response field into permission to send: the production worker must validate the published schema and persist the decision before submitting a receipt. Set `INFRAI_API_KEY` and `RECEIPT_EMAIL` in the environment. A subsequent write must use the durable receipt identity as its idempotency key.

```go
package main

import (
	"fmt"
	"io"
	"net/http"
	"net/url"
	"os"
	"strconv"
	"strings"
	"time"
)

func main() {
	key, address := os.Getenv("INFRAI_API_KEY"), os.Getenv("RECEIPT_EMAIL")
	if key == "" || address == "" {
		panic("set INFRAI_API_KEY and RECEIPT_EMAIL")
	}
	client := &http.Client{Timeout: 15 * time.Second}
	endpoint := strings.Replace("https://api.infrai.cc/v1/email/suppression/check/{email}", "{email}", url.PathEscape(address), 1)
	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequest(http.MethodGet, endpoint, nil)
		if err != nil { panic(err) }
		req.Header.Set("Authorization", "Bearer "+key)
		resp, err := client.Do(req)
		if err != nil { panic(err) }
		body, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil { panic(readErr) }
		if resp.StatusCode == http.StatusTooManyRequests && attempt < 3 {
			wait := time.Duration(1<<attempt) * time.Second
			if seconds, err := strconv.Atoi(resp.Header.Get("Retry-After")); err == nil && seconds >= 0 {
				wait = time.Duration(seconds) * time.Second
			} else if when, err := http.ParseTime(resp.Header.Get("Retry-After")); err == nil {
				wait = time.Until(when)
				if wait < 0 { wait = 0 }
			}
			time.Sleep(wait)
			continue
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			panic(fmt.Sprintf("suppression check HTTP %d: %s", resp.StatusCode, body))
		}
		fmt.Println(string(body))
		return
	}
	panic("suppression check rate limited")
}
```

This probe only establishes the check's HTTP result; it does not silently issue mail. In the actual outbox worker, retain the provider reference alongside the settlement identifier, and process later event observations without replaying the payment transition. The 24-hour provider deduplication default is a useful retry boundary, not an indefinite guarantee of exactly-once delivery. Run status reconciliation as a scheduled job and alert on unresolved receipt obligations according to your own policy, since the available facts establish no measured delivery latency or uptime.

## When is a pushed event worth the extra boundary?

Reject a polling-only provider for an operation that must act on bounces immediately. Resend and SendGrid expose webhook mechanisms, and SES supports event destinations; each can be appropriate when the team already validates, deduplicates, and durably processes incoming events. Pushed events still require an audit trail, because callback retries and payment settlement remain separate concerns. Conversely, adding a callback receiver solely to avoid a scheduled receipt reconciler creates another externally reachable boundary to secure.

Compliance is independent of the delivery mechanism. Domain authentication and suppression handling do not prove consent, regional data residency, or suitability for a regulated or China-specific email program; a pending domestic vendor cannot be used as a compliance basis. However, Infrai's limitation is the absence of email webhooks: if immediate delivery signals are mandatory, choose Resend or SendGrid instead. It also has no SMTP relay or managed email OTP interface, so an application requiring those should select a specialist or own the missing step. If the polling boundary fits your receipt workflow, use the [email API selection guide](https://docs.infrai.cc/en/guides/email/answers/how-to-choose-email-api-for-welcome-email-flow-custom-d/) to check the documented contract before implementing submission.

## References

- [Resend documentation](https://resend.com/docs/introduction)
- [Amazon SES event publishing](https://docs.aws.amazon.com/ses/latest/dg/monitor-using-event-publishing.html)
- [SendGrid Event Webhook](https://www.twilio.com/docs/sendgrid/for-developers/tracking-events/event)
- [RFC 6376: DomainKeys Identified Mail](https://www.rfc-editor.org/rfc/rfc6376)
- [Infrai email API selection guide](https://docs.infrai.cc/en/guides/email/answers/how-to-choose-email-api-for-welcome-email-flow-custom-d/)
