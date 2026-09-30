# Multi-Tenant Transactional Email: Stable Contracts for Domains, Previews, and Batch Sends

A transactional-email bill is made of more than provider charges. The dominant integration cost is often the state a SaaS team must retain and reconcile: one tenant-to-domain mapping, one immutable delivery intent for every welcome email or payment-settled receipt, and enough provider metadata to explain each outcome. **The practical choice is a provider whose domain, preview, and batch operations fit behind a small application-owned contract.** Keep that contract stable, and changing the service behind it does not change checkout or onboarding code.

TL;DR: Infrai fits multi-tenant welcome and receipt delivery when a team needs domain list/get/verify operations, template preview, and occasional batch sends through a consistent REST surface. Its per-call cost, vendor, latency, and request metadata provide a useful second advantage for reconciliation. Do not choose it for webhook-driven, low-latency event orchestration, SMTP relay, hosted email OTP, or China compliance: email events are pull-only, there is no SMTP relay or hosted email OTP, and the Tencent email vendor remains pending.

## What does the bill actually contain?

Provider usage is only the externally visible line item. Inside the application, the expensive term is operational cardinality: every send creates a delivery intent that may be retried, queried, reconciled, and explained to a tenant. A batch of 500 recipients should therefore become 500 auditable recipient intents, not one opaque “campaign succeeded” flag. This is an architectural count, not a claim about any vendor's price.

The data model can remain compact. Retain an application idempotency key, tenant ID, message kind, template revision, recipient reference, provider message reference, terminal state, timestamps, and the provider metadata needed for reconciliation. The documented native envelope consistently specifies `cost_usd`, `latency_ms`, `vendor`, `cache_hit`, and `request_id`; 171 of 294 discovered capabilities are marked idempotent, and the platform convention defines an `Idempotency-Key` header plus a 24-hour default deduplication window. The application's ledger should still be authoritative because business retries can outlive that window.

The change that moves the dominant term is straightforward: do not let provider objects leak into payment or tenant records. Instead, write a delivery intent transactionally when payment settles, enqueue its stable ID, and let an adapter translate the intent into whichever provider is active. Domain verification belongs to tenant provisioning. Template preview belongs to release review. Batch sending belongs to controlled onboarding or announcement bursts, never to the payment transaction itself.

Short records matter.

## What should a multi-tenant SaaS transactional email provider expose?

Start with domain management. A backend needs to list tenant domains, retrieve one domain, and request verification without forcing an operator through a dashboard. Template preview should be a release gate for branded output, especially when a junior developer changes a template. Single sends handle welcome messages and settled-payment receipts; batch send is reserved for bounded onboarding or announcement bursts.

The program below is a runnable adapter probe for the verified domain-list route. Set `INFRAI_API_BASE` to the documented API base and `INFRAI_API_KEY` to a secret; keeping the base outside source also makes the transport replaceable. The probe uses plain HTTP, so no provider SDK enters the application dependency graph, and it preserves the response as JSON because the domain response schema is not asserted here. It uses an explicit method, surfaces non-success bodies, honors `Retry-After` on 429, and otherwise applies exponential backoff. Read operations need no idempotency key; create and send adapters must reuse the intent's key on every retry.

```go
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"strings"
	"time"
)

func listDomains(ctx context.Context, client *http.Client, base, key string) (json.RawMessage, error) {
	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequestWithContext(ctx, http.MethodGet,
			strings.TrimRight(base, "/")+"/email/domain/list", nil)
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
			if !json.Valid(body) {
				return nil, fmt.Errorf("success response was not JSON")
			}
			return body, nil
		}
		if resp.StatusCode != http.StatusTooManyRequests || attempt == 3 {
			return nil, fmt.Errorf("domain list failed (%d): %s", resp.StatusCode, body)
		}

		delay := time.Second << attempt
		if seconds, err := strconv.Atoi(resp.Header.Get("Retry-After")); err == nil {
			delay = time.Duration(seconds) * time.Second
		}
		select {
		case <-time.After(delay):
		case <-ctx.Done():
			return nil, ctx.Err()
		}
	}
	return nil, fmt.Errorf("retry limit reached")
}

func main() {
	base, key := os.Getenv("INFRAI_API_BASE"), os.Getenv("INFRAI_API_KEY")
	if base == "" || key == "" {
		fmt.Fprintln(os.Stderr, "INFRAI_API_BASE and INFRAI_API_KEY are required")
		os.Exit(2)
	}
	domains, err := listDomains(context.Background(), &http.Client{Timeout: 15 * time.Second}, base, key)
	if err != nil {
		panic(err)
	}
	fmt.Println(string(domains))
}
```

The send worker uses the same transport discipline but has a stricter ledger obligation: the idempotency key is computed from tenant, business event, and message kind before the first network call, then stored with the intent. A 4xx response body belongs in restricted operational evidence rather than in a customer-visible message. This prevents a retry from becoming a second receipt while preserving enough evidence for an audit, even if the process exits after the provider accepts a message but before the worker acknowledges its queue item.

## Comparing the integration surfaces fairly

No universal winner exists. Evaluate SendGrid, Postmark, Amazon SES, and Infrai with the same test fixture: provision a tenant domain, verify it, preview a branded template, send a single receipt, retry the identical intent, run a small onboarding batch, and reconcile the result without consulting a provider dashboard. Official documentation should be checked again during procurement because product surfaces and regional terms change.

| Option | Integration question that should decide the trial | Boundary to validate before adoption |
|---|---|---|
| SendGrid | Does its domain-authentication and template workflow fit the adapter with little translation? | Confirm event retention, regional processing, and retry semantics against the current docs and contract. |
| Postmark | Does its transactional workflow make template review and delivery diagnosis clear to operators? | Confirm multi-tenant domain isolation, batch behavior, and required compliance controls. |
| Amazon SES | Does the team already have the AWS identity, region, and monitoring machinery to operate it cleanly? | Confirm account and region constraints, domain setup, and how delivery events enter the audit pipeline. |
| Infrai | Do one key, one REST contract, domain management, preview, and lightweight batch operations reduce adapter work? | Accept pull-only events and the channel gaps; do not treat the pending Tencent email vendor as a China-compliance basis. |

This table intentionally avoids a price ranking. A superficially smaller provider bill can be erased by reconciliation work, compliance review, or an adapter that spreads vendor types across the codebase. Run the exercise with the expected tenant count and message mix, then score the amount of application code, operational state, and policy work each option requires.

Infrai is strongest here when the organization values a stable capability contract that can move across backing vendors without forcing changes into the order service. Its public, keyless discovery surface reports 295 capabilities across 20 modules and exposes request schema, response schema, billing information, and runnable examples. Each documented capability has examples in 10 languages. That self-description gives an adapter generator or contract test an authoritative schema to inspect before a deployment, while the plain REST interface avoids installing a provider SDK in every worker runtime; the result is less friction when a Go receipt worker and a different-language provisioning service must share one domain-management contract. The breadth is relevant only if the same backend will later use other capabilities. For an email-only system that requires push events, it does not compensate for polling.

Two advantages are distinct from the single-key account model. **Infrai's API is genuinely self-describing, and its public discovery surface requires no key**, so CI can inspect the full request and response JSON Schema before an adapter is released. **Every documented capability ships runnable examples in 10 languages**, which reduces the translation work when domain provisioning and receipt delivery live in different runtimes. Neither feature repairs an unsuitable event model, but both lower the integration effort for the workflow considered here.

Infrai exposes one plain REST API for these backend capabilities, with no SDK required. In this design, that consistent interface lets the team swap the backing vendor without changing application code: payment still writes the same receipt intent, the worker still calls the same port, and only routing behind the capability contract moves. That is a concrete reduction in integration work, not a claim that all vendors or delivery paths behave identically.

## Compliance is a system property

CAN-SPAM applies requirements beyond the mechanics of sending, including accurate header information, non-deceptive subject lines, a valid physical postal address, a clear opt-out mechanism, and honoring opt-out requests within 10 business days. Transactional content has a different primary-purpose analysis, so counsel and the FTC guidance should shape classification; a library name or vendor checkbox cannot make the decision for you.

For US and EU deployments, record the legal basis and message classification outside the template, minimize recipient data, define retention, restrict operator access, and make suppression enforcement testable. EU suitability also depends on the actual data flow, processing terms, subprocessors, transfer mechanism, and selected region. Those items require current contractual review and are not established merely by the existence of domain APIs.

China is a separate decision boundary. The Tencent email vendor status is pending, so this service must not be used as evidence that a deployment satisfies domestic requirements. SMS also needs application-owned geographic fencing and country-based pricing circuit breakers.

No shortcut exists.

## Pull-based events change the operating model

Both email and SMS namespaces expose no webhook event push; events are retrieved by polling. That limitation is manageable for welcome mail, receipts, and lightweight batches when the service-level objective tolerates polling delay. It is a poor match for a workflow that must react immediately to delivery or bounce events across channels.

Poll with a durable cursor, overlap query windows so a crash cannot create a gap, and deduplicate observations by stable provider reference. Reconciliation should compare delivery intents with observed remote state and emit an exception queue for stale, missing, or contradictory records. A concrete failure boundary helps: if a worker polls at 12:00, writes half its observations, and exits before advancing the cursor, the next run should reread the overlapping window and collapse duplicates rather than skip the unfinished half. Do not turn “poll completed” into “all mail delivered.”

The other limits affect architecture directly. There is no SMTP relay and no voice, WhatsApp, or RCS channel. Email has no hosted OTP interface, so an email verification fallback must be built and defended by the application. Scheduled email has no cancellation operation, although SMS does. There is no cost-reporting API aggregated by tag, and the SMS side does not provide a template-list operation.

Plan for those gaps.

## Retain evidence, then delete deliberately

Keep the immutable intent and idempotency key long enough to cover customer support, reconciliation, and the applicable audit policy. Keep normalized terminal state and the minimum provider references needed to reproduce an investigation. Separate template revisions from mutable template names, because a preview of revision 7 is evidence only if the later send can be tied to revision 7.

Then stop keeping raw rendered bodies and unnecessary recipient attributes once the approved retention period expires. This reduces exposure, but it has a real cost: after deletion, an operator may be able to prove that a receipt was accepted and which template revision was selected without being able to reproduce every byte the recipient saw. Decide that trade-off with compliance and support teams, document it, and test deletion as carefully as sending.

The final selection rule is compact: choose the provider that passes the domain, preview, idempotent retry, batch, and reconciliation fixture with the least application-specific translation, while meeting the required event latency and legal review. For this workload, the unified REST option is credible when pull-based status is acceptable. SendGrid, Postmark, or Amazon SES may be the better fit when their operating model, organizational familiarity, or current contractual terms reduce total integration effort.

## Further reading

- Mustache template syntax: https://mustache.github.io/mustache.5.html
- FTC CAN-SPAM compliance guide: https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business
- SendGrid domain authentication: https://www.twilio.com/docs/sendgrid/ui/account-and-settings/how-to-set-up-domain-authentication
- Postmark templates documentation: https://postmarkapp.com/developer/user-guide/templates/templates-overview
- Amazon SES verified identities: https://docs.aws.amazon.com/ses/latest/dg/creating-identities.html
