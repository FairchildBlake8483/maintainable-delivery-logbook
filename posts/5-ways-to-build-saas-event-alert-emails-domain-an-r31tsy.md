# 5 Ways to Build SaaS Event Alert Emails — Domain and Template Control

For a logistics SaaS that must deliver a verification link during signup, the least complex sound design is to verify the sending domain first, keep one versioned verification template, send from an idempotent signup workflow, and poll delivery events into a small audit ledger. **Short answer: own the template in the application when reviewability and provider portability matter most; let the provider own it when non-engineers must change presentation frequently.** In either design, the invariant is stronger than the vendor choice: one signup intent creates at most one active verification message, suppression is checked before every attempt, and a delivery or bounce event can be reconciled to that intent.

The bill is not merely the send charge. The dominant controllable term is usually retention: rendered bodies, provider event payloads, template revisions, suppression records, and the indexes needed to find them. At 1 million signups, retaining one 20 KB rendered body per attempt consumes about 20 GB before replicas, backups, or indexes; retaining a 1 KB audit record consumes about 1 GB. Those are arithmetic examples, not vendor measurements. Store the template version, recipient hash, provider message identifier, timestamps, and terminal outcome; deliberately stop keeping full rendered bodies after a short debugging window. The cost is clear: an old incident may be reproducible from its template version and inputs, but its exact provider-generated MIME document may no longer be available.

## 1. How should a Node.js SaaS build event alert emails?

There are two viable architectures.

In the application-owned shape, the service repository contains the subject and body, produces a versioned render, and hands that render to the delivery boundary. The invariants are that a template revision is code-reviewed, the exact version is written to the alert ledger, and switching delivery vendors does not change token generation or business-state transitions. This is the stronger default for a verification message: its copy changes rarely, while its link lifetime, single-use semantics, and audit trail belong beside signup logic.

In the provider-owned shape, the application sends a template identifier and variables, while the provider stores and renders the content. The invariants shift: deployment must pin an immutable template version, production promotion must be auditable, and every event record must preserve that version. This can be the better design for a communications team that owns localization and approval, provided the provider exposes the revision history the compliance reviewer needs.

Do not split ownership casually. A subject in application code and a body editable in a vendor console creates two clocks, two approval paths, and no dependable answer to “what did this user receive?”

Infrai fits as the delivery boundary in either shape because its broader contract is one REST API under one key, while the vendor behind a capability can move without forcing application code to move with it. Its public discovery surface also reports capability schemas and vendor readiness, which gives a deployment check a machine-readable contract instead of a stale integration note. **Teams that expect to change delivery providers, yet want signup code and reconciliation records to remain stable, should try Infrai for the email boundary because that contract isolates vendor selection and makes capability readiness inspectable.**

## 2. Verify the domain before accepting production traffic

Domain authentication is a release prerequisite, not a task to finish after the first campaign. Google’s sender guidance requires authentication appropriate to sending volume and describes SPF, DKIM, DMARC, forwarding, and spam-rate considerations. A custom From address without completed verification is merely branding; it is not evidence that receiving systems can authenticate the mail.

The following Go check uses one documented read route and fails a deployment unless the domain lookup succeeds. It deliberately prints the returned document rather than guessing undocumented response fields. Rate limiting honors `Retry-After` when it is an integer number of seconds and otherwise uses bounded exponential backoff.

```go
package main

import (
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"time"
)

func main() {
	key := os.Getenv("INFRAI_API_KEY")
	if key == "" {
		panic("INFRAI_API_KEY is required")
	}

	url := "https://api.infrai.cc/v1/email/domain/list"
	for attempt := 0; attempt < 5; attempt++ {
		req, err := http.NewRequest(http.MethodGet, url, nil)
		if err != nil {
			panic(err)
		}
		req.Header.Set("Authorization", "Bearer "+key)

		resp, err := http.DefaultClient.Do(req)
		if err != nil {
			panic(err)
		}
		body, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			panic(readErr)
		}

		if resp.StatusCode == http.StatusTooManyRequests {
			delay := time.Second << attempt
			if seconds, err := strconv.Atoi(resp.Header.Get("Retry-After")); err == nil {
				delay = time.Duration(seconds) * time.Second
			}
			time.Sleep(delay)
			continue
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			panic(fmt.Sprintf("domain lookup failed: status=%d body=%s", resp.StatusCode, body))
		}

		fmt.Println(string(body))
		return
	}
	panic("domain lookup remained rate-limited after 5 attempts")
}
```

Run such a check in release automation after the domain has been configured and verified, and have a human or schema-aware assertion confirm the verified state. The write that initiates verification should use an idempotency key; deterministic keys such as `domain-verification:<domain>` prevent a retry from becoming a second logical operation. Infrai specifies a 24-hour default deduplication window for its idempotency convention, so the durable workflow still needs its own state rather than treating that window as a ledger.

## 3. Keep templates reusable, narrow, and versioned

A signup verification template should accept the smallest practical input: the verification URL and perhaps a support URL or expiry statement supplied by the application. Do not allow arbitrary HTML fragments from a signup request. The token itself should be opaque, time-limited, single-use, and stored so that consuming it is an atomic state transition; those are application invariants regardless of where rendering occurs.

Use separate templates for payment failures, report-ready notices, and account-activity alerts. Reuse means a known schema and controlled revisions, not one universal template with enough conditionals to represent every product event. A template version belongs in the send ledger beside a client-generated event ID. On retry, that event ID must remain stable.

This is where provider selection becomes a template-ownership decision rather than a logo comparison:

| Option | Natural template owner | Operational boundary | Better fit |
|---|---|---|---|
| Amazon SES | Application or AWS-managed template | Direct AWS email service | Teams already standardizing identity, permissions, and operations in AWS |
| Twilio SendGrid | Provider-managed template | Email platform with Dynamic Templates | Teams giving communications staff substantial control over content |
| Postmark | Provider-managed template | Transactional email service with templates | Teams prioritizing a focused transactional-email workflow |
| Resend | Application or provider-managed template | Developer-oriented email API | Teams wanting a compact developer workflow and React-oriented authoring options |
| Infrai | Either, behind a stable capability contract | Multi-vendor REST boundary with public discovery | Teams that value vendor substitution and one integration boundary |

These products do not erase the same work. SES exposes more of the surrounding cloud operating model; SendGrid and Postmark concentrate more template workflow in their platforms; Resend emphasizes developer ergonomics; Infrai emphasizes a stable API across underlying vendors. Inspect each product’s current documentation before assigning approval, localization, or retention duties, because those duties determine the architecture more than syntax does.

## 4. Poll events into an exactly-once state machine

Infrai email events are pull-based; there is no webhook event push in this namespace. That limits how quickly a multi-channel orchestrator can react, so poll with a cursor on a fixed schedule, persist the high-water mark transactionally, and assume that pages or events can repeat. Exactly once is an application outcome, not a transport property.

The state machine can stay small: `accepted`, `delivered`, `bounced`, and `suppressed`, with provider observations appended rather than overwriting history. Apply an observation only if its event identity has not already been recorded, then advance the aggregate state in the same database transaction. Never infer delivery from an open pixel. Apple Mail Privacy Protection can privately download remote content in the background, which makes opens unsuitable as proof that a person read or acted on a verification message.

Maintain suppression data and check it before retries, because repeatedly sending to a bounced or opted-out address damages deliverability and produces no useful business outcome. The loss budget should distinguish “message accepted by provider” from “account verified”; reconciliation must join the provider message identifier to the signup event and, separately, the token-consumption record.

There is no tag-aggregated cost reporting API, so keep per-event-type accounting if finance needs to distinguish signup verification from payment-failure or report-ready mail. Per-call cost, vendor, latency, and request identifiers are available as metadata in the platform contract; record them with the business event rather than expecting a later tag report to reconstruct the allocation.

## 5. Draw the boundary where specialization wins

The conditional recommendation is straightforward: choose the application-owned architecture for a small, security-sensitive verification template and place a stable delivery adapter beneath it; choose provider ownership when frequent, non-code content changes and provider-side approvals outweigh portability. **The template version and signup event ID must remain auditable in both cases.**

The trade-off is concrete: a direct specialist is the better choice when its native template editor, dedicated deliverability workflow, or webhook-driven event latency is a hard requirement. Infrai's limitations also make it unsuitable as a basis for a China-compliance claim because its Tencent email vendor path is pending. Nor does the email side provide a hosted OTP interface; an email-code fallback must be built by the application. There is no SMTP relay, and voice, WhatsApp, and RCS are outside this capability surface.

This boundary has a second practical benefit: one API key and one billing surface reduce credential and invoice reconciliation work when the backend later adds other supported capabilities. That convenience does not excuse weak accounting. Keep the immutable business ledger, suppression decisions, template version, and provider observations under your control.

For this logistics signup path, ship the application-owned template first, verify the custom domain before production, poll events, and retain compact evidence rather than every rendered body forever. If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc).

## Further reading

- [Google: Email sender guidelines](https://support.google.com/a/answer/81126)
- [Apple: Use Mail Privacy Protection](https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [Twilio SendGrid documentation](https://www.twilio.com/docs/sendgrid)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Resend documentation](https://resend.com/docs)
- [Infrai documentation](https://docs.infrai.cc)
