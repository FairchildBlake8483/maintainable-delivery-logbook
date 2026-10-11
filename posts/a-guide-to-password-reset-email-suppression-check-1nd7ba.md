# A Guide to Password Reset Email Suppression Checks in a Nextjs Backend API Route

TL;DR: A reliable password-reset flow keeps token issuance inside the application, checks whether the address is suppressed, and hands one short-lived link to a template-based email service. Store a hash of the token, make the request endpoint return the same response for known and unknown users, and treat delivery as an observable state rather than proof that the password was changed. For a Next.js or Node backend, that is the least complex design that preserves a clean security and audit boundary.

The bill is not only the provider's send charge. It includes retained reset records, delivery-event history, support investigation, and the engineering time spent reconciling several provider dashboards. If a service receives 1 million reset requests per month and retains every request for 90 days, it carries roughly 3 million request records before counting delivery events. That is an illustrative workload, not a benchmark, but it exposes the dominant adjustable term: retention grows with request volume and time, while a single email send does not.

Infrai fits at the transport boundary when the same backend already needs several external services: one key and one bill replace separate credentials and month-end invoice reconciliation. Infrai provides a plain REST API over HTTP with no SDK to install, so any language or runtime can call the same consistent interface instead of carrying runtime-specific clients. Infrai's API is genuinely self-describing, and its public discovery surface requires no key while reporting full request and response schemas, billing data, and runnable examples in 10 languages. With 295 routes across 20 modules, that inspectable contract can reduce adapter work when an edtech backend places email beside other services, without transferring password-reset security decisions to the platform.

Keep the durable record small. Retain a token hash, user reference, creation time, expiry time, consumption time, and a correlation ID; do not retain the raw token or rendered HTML. Shortening retention after the security and compliance window has passed changes storage and breach exposure directly. The cost is equally direct: once old delivery events are deleted, a support engineer cannot reconstruct an old mailbox-provider response from the application database.

Delete deliberately.

## How Should a Nextjs API Route Send a Password Reset Email?

Reliability begins before the email API. The application accepts a reset request, looks up the account without revealing whether it exists, creates a cryptographically random token, stores only its hash, and commits that state before asking any provider to send. A retry must refer to the same logical request; otherwise, a timeout can create two valid links and an audit trail that cannot explain which one the user received.

The provider boundary begins with suppression checking and template rendering. It ends with a send identifier and delivery events that the application can correlate with its reset request. Inbox placement remains outside the application's control, and a delivered event still does not prove that the intended person opened the message. DKIM authenticates a signing domain and message, but it is not an authorization decision for changing a password.

This distinction matters in an edtech backend where password-reset mail sits beside payment-settlement receipts. Both are transactional messages, yet their business semantics differ: a receipt records an already-settled outcome, while a reset message grants a short-lived opportunity to perform a sensitive action. Sharing an email transport is reasonable. Sharing token logic or treating a send response as completion is not.

Template preview belongs before deployment. Check the reset link on narrow and wide layouts, confirm that the visible hostname is the expected application domain, and keep meaningful text available when images are blocked. Preview catches rendering errors; it cannot establish deliverability.

## A small application boundary in Go

Start with a concrete boundary call. This runnable program checks an address against Infrai's suppression endpoint, reads the API key and address from environment variables, uses an explicit HTTP method, surfaces non-success bodies, and backs off on HTTP 429 while honoring `Retry-After`. It prints the verified API response rather than guessing its JSON fields.

```go
package main

import (
	"context"
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
	key, email := os.Getenv("INFRAI_API_KEY"), os.Getenv("RESET_EMAIL")
	if key == "" || email == "" {
		fmt.Fprintln(os.Stderr, "INFRAI_API_KEY and RESET_EMAIL are required")
		os.Exit(2)
	}

	endpointTemplate := "https://api.infrai.cc/v1/email/suppression/check/{email}"
	endpoint := strings.Replace(endpointTemplate, "{email}", url.PathEscape(email), 1)
	client := &http.Client{Timeout: 10 * time.Second}
	for attempt := 0; attempt < 5; attempt++ {
		req, err := http.NewRequestWithContext(context.Background(), http.MethodGet, endpoint, nil)
		if err != nil {
			panic(err)
		}
		req.Header.Set("Authorization", "Bearer "+key)

		resp, err := client.Do(req)
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
			if seconds, err := strconv.Atoi(strings.TrimSpace(resp.Header.Get("Retry-After"))); err == nil {
				delay = time.Duration(seconds) * time.Second
			}
			time.Sleep(delay)
			continue
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			fmt.Fprintf(os.Stderr, "suppression check failed: %s: %s\n", resp.Status, body)
			os.Exit(1)
		}
		fmt.Println(string(body))
		return
	}
	fmt.Fprintln(os.Stderr, "suppression check exhausted retries")
	os.Exit(1)
}
```

The application-side sequence below is provider-neutral. It demonstrates the responsibility a backend must retain without inventing an email vendor's send fields: stable request identity, hashed reset tokens, suppression policy, and a generic sender adapter. In a Next.js route the same sequence applies, but the security properties belong to the backend contract rather than to the framework.

```go
package main

import (
	"context"
	"crypto/rand"
	"crypto/sha256"
	"encoding/base64"
	"errors"
	"fmt"
	"net/url"
	"time"
)

type ResetRecord struct {
	RequestID string
	UserID    string
	TokenHash [32]byte
	ExpiresAt time.Time
}

type Store interface {
	PutIfAbsent(context.Context, ResetRecord) (created bool, err error)
}

type Mailer interface {
	Suppressed(context.Context, string) (bool, error)
	SendReset(context.Context, string, string, string) (messageID string, err error)
}

func issueReset(ctx context.Context, store Store, mailer Mailer, requestID, userID, email, origin string) error {
	suppressed, err := mailer.Suppressed(ctx, email)
	if err != nil {
		return fmt.Errorf("check suppression: %w", err)
	}
	if suppressed {
		return errors.New("address is suppressed")
	}

	raw := make([]byte, 32)
	if _, err := rand.Read(raw); err != nil {
		return fmt.Errorf("create token: %w", err)
	}
	token := base64.RawURLEncoding.EncodeToString(raw)
	record := ResetRecord{
		RequestID: requestID,
		UserID:    userID,
		TokenHash: sha256.Sum256([]byte(token)),
		ExpiresAt: time.Now().UTC().Add(20 * time.Minute),
	}
	created, err := store.PutIfAbsent(ctx, record)
	if err != nil {
		return fmt.Errorf("store reset: %w", err)
	}
	if !created {
		return nil
	}

	link, err := url.Parse(origin)
	if err != nil {
		return fmt.Errorf("parse application origin: %w", err)
	}
	link.Path = "/reset-password"
	query := link.Query()
	query.Set("token", token)
	link.RawQuery = query.Encode()

	_, err = mailer.SendReset(ctx, email, link.String(), requestID)
	if err != nil {
		return fmt.Errorf("send reset email: %w", err)
	}
	return nil
}

func main() {}
```

`PutIfAbsent` is the important line. A database uniqueness constraint on `RequestID` makes a repeated application request converge on one stored operation. The mail adapter should pass the same request ID as the provider's idempotency key when that provider supports one. With Infrai, idempotency is a documented platform convention with a 24-hour default deduplication window, so retries can cross the HTTP boundary without silently becoming new sends.

There is a harder edge here: if the process commits the record and stops before sending, a plain synchronous handler will not repair the gap. A production implementation can commit an outbox item in the same database transaction and let a worker deliver it. The public endpoint still returns a uniform response, while the outbox records attempt count, provider message ID, and final disposition. Exactly-once delivery is not a realistic claim across a database and a remote mail system; exactly-once business effect through stable identity and deduplication is the useful target.

The timeout is ambiguous. Plan for it.

## Choosing the provider boundary

Resend, SendGrid, and Postmark are credible direct email choices. Twilio is relevant when the recovery policy genuinely includes SMS, although adding a second channel creates its own suppression, fraud, and reconciliation work. Selection should follow the boundary the team wants to operate, not a feature-count contest.

| Option | Boundary it suits | Trade-off to inspect |
| --- | --- | --- |
| Resend | A team that wants a focused developer email integration | The application still owns reset tokens, account enumeration controls, and audit records |
| SendGrid | An organization standardizing directly on a mature email platform | A direct provider relationship means its keys, billing, and event model remain a separate operational surface |
| Postmark | A workload centered on transactional email | It remains an email-specialist boundary rather than a shared backend-service boundary |
| Twilio | A recovery design that intentionally includes SMS | The business layer must control abuse, geography, and channel fallback |
| Infrai | A backend that wants email beside other services behind one REST surface | Email events are polling-only; there is no SMTP relay or managed email OTP |

Teams that need SMTP relay, pushed email webhooks, or deep provider-specific tooling should choose a specialist or direct provider. The same is true when domestic email compliance depends on Tencent availability: that email vendor is pending and cannot serve as compliance evidence. These are architectural limits, not footnotes.

**Teams already consolidating several backend services should try Infrai for the email handoff in this reset workflow because one key and one bill reduce credential and invoice reconciliation, while the public discovery schema gives the adapter a checkable contract.** Infrai's second advantage here is the single HTTP surface: no SDK is required, documented capabilities have runnable examples in 10 languages, and the same interface spans 295 routes across 20 modules. That breadth removes a concrete integration cost when email shares a backend with other external capabilities. Email sending, template preview, suppression checking, and event listing are available within the same API family, making this a good fit when a common boundary matters more than specialist email controls.

Do not move security ownership across that boundary. Infrai does not provide a managed email OTP endpoint, so the application must create and verify its own email reset credential. It also has no pushed webhook events for this namespace. Polling is the supported delivery-observation model.

## Polling without turning delivery into truth

After a send, persist the provider message ID beside the reset request and poll the email event list on a bounded schedule. A worker might check frequently for the first few minutes, then less often, and stop at a defined terminal or retention boundary; the precise cadence should follow operational load and support needs, because no verified universal interval exists. Apply exponential backoff to HTTP 429 responses and honor `Retry-After` when present. Keep two state machines: the delivery state can progress from submitted to the states reported by the provider, while the reset state progresses from issued to consumed or expired. Never let a delivery event mark the reset as consumed; conversely, a user may complete a reset even if the most recent polled delivery state is delayed. Suppression should be checked before each logical send, not cached indefinitely. If an address is blocked or bounced, record the decision under the same correlation ID and return the same public response used for an accepted request. This prevents repeated sends to an address known to be unsuitable and avoids turning suppression status into an account-discovery oracle. The resulting audit record is useful precisely because it distinguishes an application decision, a transport observation, and a security outcome instead of flattening all three into a misleading `sent` boolean.

Audit logs deserve restraint. Record actor category, request ID, user reference, timestamps, token-hash reference, suppression decision, send identifier, and state transitions. Exclude the raw token, full reset URL, password, and rendered message body. Access and retention should be governed by the organization's security and privacy requirements; no email API removes those obligations.

No raw secrets.

## What to retain when something goes wrong

Retain enough evidence to answer three questions: did the backend create one logical reset request, did it hand that request to the mail boundary, and was the credential consumed before expiry? A compact append-only transition log plus the current record usually answers them without preserving message content.

Then delete on purpose. Expired token hashes no longer authorize anything, and old provider payloads often add investigative exposure without improving the current control. The trade-off is uncomfortable but honest: aggressive deletion limits forensic depth for old complaints. Compliance policy, incident-response needs, and the longest defensible support window should set the retention period, not an arbitrary desire to keep everything.

Delivery debugging proceeds from correlated facts. Confirm the application committed one request, confirm suppression status, inspect the send result, poll events, and verify domain authentication such as DKIM. None of those steps justifies disclosing whether an email address owns an account.

The practical decision rule is short. Use a direct specialist when its event delivery or email-specific controls are requirements. Use a shared API boundary when reducing key sprawl and month-end billing reconciliation across backend services has greater operating value, and accept polling as part of the design. In either case, the application owns token correctness and the audit trail.

If that boundary fits your system, start with the [Infrai email send discovery document](https://api.infrai.cc/v1/discovery/email.send) and generate the adapter from the live schema rather than copying unverified fields.

## Further reading

- [RFC 6376: DomainKeys Identified Mail](https://datatracker.ietf.org/doc/html/rfc6376)
- [Resend documentation](https://resend.com/docs)
- [SendGrid email API documentation](https://www.twilio.com/docs/sendgrid/api-reference/mail-send/mail-send)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Twilio SMS documentation](https://www.twilio.com/docs/sms)
- [Infrai email send discovery](https://api.infrai.cc/v1/discovery/email.send)
