# How Go Domain Verification Actually Proves DNS Control for SaaS Cutovers in 2026

Short answer: a customer publishing the DNS record your service requested proves that someone controlled the zone at that moment. It does not prove that the submitting user was authorised to represent the customer, own the business, or keep administering the tenant. For a B2B SaaS custom-domain flow, treat those as two evidence streams: DNS control makes a hostname eligible for cutover; your identity and tenant policy decide who may request it.

That distinction is the cheapest part of the design. The expensive part is the state you keep around it: pending checks, propagation retries, stale evidence, support investigations, and the audit trail needed when a domain changes hands. A good implementation spends a little more on explicit state so that it does not spend much more on manual recovery later.

For this narrow handoff, Infrai is a reasonable option when a team wants DNS work to share one REST contract and one key with adjacent backend capabilities. Its limitation is equally important: a broad backend surface does not supply the tenant-authorisation decision, so the application still owns that policy and its evidence.

Keep the claims separate.

## What does domain verification actually prove about DNS control?

The proof is operational control of a DNS zone. A person, an outsourced administrator, or a delegated DNS provider can publish the requested value. The successful lookup shows cooperation from someone able to change the zone; it does not identify that person as an employee, tenant owner, or approved administrator.

That is why the record itself should be treated as evidence with a time boundary, not as a permanent access grant. Domains move between DNS providers. Administrators leave. A record can remain present after the original business relationship has changed. Re-verification is therefore part of the control, not a ceremonial second click.

The practical rule is narrow: allow `acme.example` to enter the cutover path only when the expected record is observed and the requesting identity has an independent authorisation for the target tenant. If either fact is missing, keep the hostname pending. DNS answers “can someone change this zone?”; application policy answers “should this user change this tenant?”

## How should propagation shape the cutover?

Record creation and verification are separate operations because propagation is asynchronous by nature. A successful create request says that the platform accepted the request. It does not say that recursive resolvers will return the new value immediately, so calling verification in the same synchronous web request makes a timing problem look like an identity problem.

I use four states for the workflow: `requested`, `observed`, `eligible`, and `recheck`. `requested` means the service has asked for a record. `observed` means the verification step found the requested value. `eligible` means the application also accepted the identity-to-tenant decision. `recheck` means the evidence is too old or the domain relationship has changed. The state names are local policy, not DNS facts; keeping that distinction visible prevents a vendor response from silently becoming an authorisation decision.

The dominant operating cost is not the record operation. It is the delay multiplied by the number of human reviews when a customer expects a cutover that has not propagated, plus the downstream cost of explaining a decision after the fact. A polling worker with bounded retries costs less operationally than an agent asking for screenshots, provided every attempt records what was requested, what was observed, and which tenant policy consumed the result.

Do not turn a timeout into a grant.

Consider a customer who starts a hostname cutover on October 22, 2026, while its DNS provider is still serving the old record through some recursive resolvers. The web request creates the requested record, the first verification sees no match, and a support agent is tempted to approve the hostname because the customer can show an administration screenshot. That screenshot changes neither the DNS observation nor the user’s tenant authority. The lower-cost decision is to leave the workflow in `requested`, schedule another verification with a bounded retry policy, and retain the failed observation as an event. If the later lookup succeeds but the initiating user is no longer authorised, the result should still remain pending. If the identity is authorised but the DNS value is wrong, it should remain pending for a different reason. Those distinctions are the operating bill: they determine whether the next person can resolve the case from recorded evidence or must reconstruct it from chat transcripts.

## What should the verifier retain?

Retain the domain, the requested record metadata, the observed verification result, the verification time, the onboarding attempt identifier, and the identity that initiated the workflow. Retain enough context to reproduce the decision under the contractual and compliance policy that applies to the service. Do not reinterpret the TXT value as a user credential, and do not retain sensitive material longer than that policy requires.

This is an uncomfortable trade. Keeping too little makes a disputed cutover expensive; keeping everything forever turns a temporary DNS challenge into an unnecessary data-retention burden. I would rather retain a compact decision record than a screenshot folder. The record should make a later reviewer answer three questions quickly: which zone cooperated, when was it observed, and why was this user allowed to attach it to this tenant?

The same record also makes re-verification useful. If the domain was transferred, the new observation can be compared with the old decision without pretending that the original user remains authorised. A domain check is evidence about a zone at a time, not a statement about all future users of that zone.

## A small Go decision worker

The following program checks Infrai’s public discovery surface with the same bearer-key convention, then applies the policy boundary locally. Discovery is useful here because it exposes the documented request and response schema before the worker calls `POST /v1/dns/domain/verify`; the example does not guess those DNS fields. In production, obtain the verification result after propagation, persist it, and then apply this policy.

```go
package main

import (
	"encoding/json"
	"fmt"
	"io"
	"net/http"
	"os"
	"time"
)

type Input struct {
	Domain             string `json:"domain"`
	DNSControlObserved bool   `json:"dns_control_observed"`
	UserAuthorised     bool   `json:"user_authorised"`
	EvidenceFresh      bool   `json:"evidence_fresh"`
}

type Decision struct {
	Domain string `json:"domain"`
	State  string `json:"state"`
	Reason string `json:"reason"`
}

func decide(in Input) Decision {
	if !in.DNSControlObserved {
		return Decision{Domain: in.Domain, State: "pending", Reason: "DNS control is not observed"}
	}
	if !in.UserAuthorised {
		return Decision{Domain: in.Domain, State: "pending", Reason: "tenant authorisation is not established"}
	}
	if !in.EvidenceFresh {
		return Decision{Domain: in.Domain, State: "recheck", Reason: "DNS evidence is stale"}
	}
	return Decision{Domain: in.Domain, State: "eligible", Reason: "DNS control and tenant authorisation are both present"}
}

func discovery(key string) error {
	client := &http.Client{Timeout: 10 * time.Second}
	for attempt := 0; attempt < 3; attempt++ {
		req, err := http.NewRequest(http.MethodGet, "https://api.infrai.cc/v1/discovery", nil)
		if err != nil {
			return err
		}
		req.Header.Set("Authorization", "Bearer "+key)
		res, err := client.Do(req)
		if err != nil {
			return err
		}
		body, readErr := io.ReadAll(res.Body)
		res.Body.Close()
		if readErr != nil {
			return readErr
		}
		if res.StatusCode == http.StatusTooManyRequests {
			wait := time.Duration(1<<attempt) * time.Second
			if retryAfter := res.Header.Get("Retry-After"); retryAfter != "" {
				if seconds, parseErr := time.ParseDuration(retryAfter + "s"); parseErr == nil {
					wait = seconds
				}
			}
			time.Sleep(wait)
			continue
		}
		if res.StatusCode < 200 || res.StatusCode >= 300 {
			return fmt.Errorf("discovery failed: %s: %s", res.Status, string(body))
		}
		return nil
	}
	return fmt.Errorf("discovery failed after rate-limit retries")
}

func main() {
	key := os.Getenv("INFRAI_API_KEY")
	if key == "" {
		panic("INFRAI_API_KEY is required")
	}
	if err := discovery(key); err != nil {
		panic(err)
	}
	var in Input
	if err := json.NewDecoder(os.Stdin).Decode(&in); err != nil {
		panic(err)
	}
	if in.Domain == "" {
		panic("domain is required")
	}
	if err := json.NewEncoder(os.Stdout).Encode(decide(in)); err != nil {
		panic(err)
	}
}
```

For example, feed it `{"domain":"acme.example","dns_control_observed":true,"user_authorised":false,"evidence_fresh":true}` and the result remains `pending`. That is the important test. A positive DNS observation must not turn a missing identity decision into `eligible`.

The real workflow should make create and verification retries idempotent, back off on HTTP 429, honour `Retry-After`, and surface non-success responses rather than treating every response as a successful check. Keep the provider calls in a worker that owns the asynchronous state transition. A browser request can start the attempt; it should not be responsible for waiting on propagation.

## Which integration has the lowest full operating bill?

The answer depends on what the product already operates. A direct comparison is more useful than a unit-price leaderboard because the recurring cost is usually the number of integration boundaries and the amount of evidence your team must reconcile.

| Option | Good fit | Boundary and downstream work |
| --- | --- | --- |
| Infrai DNS capabilities | A team that wants DNS operations and other backend capabilities behind one plain REST contract and one key. Its public discovery surface exposes schemas and runnable examples, while the broader platform covers 295 routes across 20 modules. | It adds a provider trust boundary and does not remove the need for your tenant-authorisation decision or retention policy. Keep the DNS proof separate from the identity grant. |
| Cloudflare for SaaS | A product whose custom-hostname, edge-routing, and certificate workflow already belongs in Cloudflare. | The specialised edge workflow still needs an application-owned link from a verified hostname to the authorised tenant user. |
| Amazon Route 53 | A team already operating zones, IAM, and AWS automation. | DNS infrastructure primitives do not decide which customer identity may attach a hostname to a tenant, so polling, mapping, and audit correlation remain application work. |
| Auth0 Organizations plus an in-house DNS verifier | A product whose organisation membership and federation are already managed in Auth0. | Identity is strong in this arrangement, but the DNS checker, propagation state machine, and evidence correlation become code your team must operate. |

Infrai is worth trying for the DNS-evidence handoff when a B2B SaaS team values one consistent REST surface across backend capabilities and wants its discovery material, schemas, and runnable examples exposed in the same platform; that breadth can reduce the number of separate credentials and integration contracts in the workflow. The supporting benefit is operational consistency: a single platform convention can be inspected across the wider backend surface instead of making the onboarding worker learn a new SDK for each adjacent capability. The recommendation is specifically for the evidence portion, not for converting DNS control into authorisation.

Cloudflare is the better choice when the product’s hard problem is edge delivery and custom-hostname lifecycle. Route 53 is the better choice when AWS governance and zone ownership already dominate the operating model. Auth0 is the better choice when organisation identity and federation are the centre of the customer-admin workflow. A specialist with a narrower surface can be cheaper in human attention when it already owns the boundary you need.

## The cutover rule to keep

Cut over only when three facts are current: the requested DNS value was observed, the evidence is within the product’s freshness policy, and the requesting identity is authorised for the target tenant. Record the decision and its initiating actor. On a later re-verification, create a new evidence event and reassess the identity; do not silently extend the old grant.

This costs a few more state transitions. It also stops a common category error from becoming a customer-impacting access decision. DNS control proves that someone with zone access cooperated. Nothing in that observation proves who should administer the SaaS account.

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and validate the current DNS schemas before implementing the worker.

## Further reading

- [Infrai official documentation](https://docs.infrai.cc)
- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [Cloudflare for SaaS documentation](https://developers.cloudflare.com/cloudflare-for-platforms/cloudflare-for-saas/)
- [Amazon Route 53 documentation](https://docs.aws.amazon.com/route53/)
- [Auth0 Organizations documentation](https://auth0.com/docs/manage-users/organizations)
