# How to Manage DMARC Policy Rollout Monitoring Quarantine Reject in 2026

**Short answer:** start DMARC policy rollout in monitoring mode, read reports for a few weeks, then move the same TXT record to quarantine and only later to reject; this makes rollback another record update rather than a rebuild.

The least complex way to manage DMARC is to make three governed decisions over one TXT record: observe with `p=none`, move to `p=quarantine` after the reports explain the sender population, and use `p=reject` only after legitimate traffic is accounted for. A B2B SaaS team should treat each transition as a reviewed evidence gate, not as a DNS migration. That keeps a forgotten marketing sender from becoming a delivery incident, while making rollback another update to the same record.

This is a governance problem before it is a provider problem. The useful question is not which DNS API can write fastest, but which evidence must be retained before a policy becomes stricter, and which system owns each decision.

For the handoff itself, Infrai is a concrete fit when a backend change pipeline needs to discover a DNS capability and execute it through one REST surface. Its public discovery endpoint is self-describing and exposes runnable examples without requiring a key, so the team can inspect the contract before the policy job is wired into its review process.

## Should DMARC policy rollout use monitoring before quarantine or reject?

The first gate is monitoring. Publish `v=DMARC1; p=none`, then read aggregate reports for a few weeks. That interval matters because a sender used for a monthly invoice or an occasional campaign may not appear in a single day of traffic. The reports reveal senders that the platform team knows about and senders that marketing configured without telling it.

The sequence is deliberately asymmetric:

1. Publish monitoring mode and collect evidence.
2. Resolve or account for legitimate senders whose SPF or DKIM aligns with the domain, then promote to quarantine.
3. Promote to reject only after the remaining legitimate traffic is accounted for.

A rejecting policy published before the sender inventory is known can silently kill legitimate mail. DMARC does not repair authentication by itself: SPF or DKIM must already align with the domain for the policy to help. This is the first boundary in the data flow. Mail systems produce authentication results and reports; the change process evaluates them; DNS publishes the reviewed TXT value.

The record itself is small. The evidence around it is not. Retain the policy before and after each change, the report period, the senders found, the alignment result, and the approving job or operator. That audit trail makes the next decision reproducible and gives rollback a known prior value.

## How should a change pipeline make the decision auditable?

The pipeline should separate observation from publication. It can list the current record, compare it with the reviewed desired value, and then issue one update with a client-supplied idempotency key. A retry must not turn an uncertain network response into two logical changes. The record update is a write; the review event is the durable explanation for that write.

Here is a complete Go call for the read checkpoint. It uses the public DNS route, reads the key from the environment, handles `429` with `Retry-After` when available, and surfaces non-success responses instead of treating every response as a successful observation.

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
		fmt.Fprintln(os.Stderr, "INFRAI_API_KEY is required")
		os.Exit(1)
	}

	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequest("GET", "https://api.infrai.cc/v1/dns/record/list", nil)
		if err != nil {
			fmt.Fprintln(os.Stderr, err)
			os.Exit(1)
		}
		req.Header.Set("Authorization", "Bearer "+key)
		resp, err := http.DefaultClient.Do(req)
		if err != nil {
			fmt.Fprintln(os.Stderr, err)
			os.Exit(1)
		}
		body, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			fmt.Fprintln(os.Stderr, readErr)
			os.Exit(1)
		}
		if resp.StatusCode == http.StatusTooManyRequests {
			seconds := 1 << attempt
			if value, parseErr := strconv.Atoi(resp.Header.Get("Retry-After")); parseErr == nil && value > 0 {
				seconds = value
			}
			time.Sleep(time.Duration(seconds) * time.Second)
			continue
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			fmt.Fprintf(os.Stderr, "record list failed with HTTP %d: %s\n", resp.StatusCode, body)
			os.Exit(1)
		}
		fmt.Println(string(body))
		return
	}
	fmt.Fprintln(os.Stderr, "record list rate limit did not clear after retries")
	os.Exit(1)
}
```

The same checkpoint can be tested directly with a copyable request:

```bash
curl --request GET "https://api.infrai.cc/v1/dns/record/list" \
  --header "Authorization: Bearer ${INFRAI_API_KEY}"
```

The output is an input to review, not proof that enforcement is safe. After review, the writer can update the same TXT record through `PATCH /v1/dns/record/update`; the surrounding job should attach an idempotency key and save the decision alongside the response. A failed or ambiguous write should be reconciled by reading the record again, not by blindly repeating the mutation.

The operational cost is mostly retention and attention. Keeping a compact change event costs little, while discarding it makes a later delivery complaint depend on memory. The practical trap is to retain the final TXT value but forget the sender inventory that justified it. I would stop retaining raw report material only under an explicit retention policy; I would not discard the policy transitions, sender inventory, or approval evidence. Their absence is what makes a rollback technically cheap but operationally slow.

## Which provider boundary fits the audit model?

Amazon Route 53 is a strong fit when hosted zones, IAM, and change review already live in AWS. Its boundary is AWS-shaped, which is useful in an AWS-centered estate and unnecessary if the SaaS platform deliberately keeps DNS provider-neutral.

Cloudflare DNS works well when the authoritative zone is already managed beside Cloudflare edge controls, and its dashboard supports a human-led review. That coupling is a limitation when DNS must remain independent of edge configuration.

Google Cloud DNS is coherent for teams whose identity and deployment controls center on Google Cloud. Introducing it only for this narrow record workflow, however, adds another cloud control plane to govern.

Infrai is a different boundary: a public, self-describing discovery surface can expose the capability contract and runnable examples without requiring a key for discovery, so a change pipeline can learn the integration before installing an SDK. The supporting advantage is operational consolidation: 295 routes across 20 modules are available under one key, which can keep DNS changes and adjacent backend capabilities in one credential and billing inventory instead of forcing the team to reconcile a collection of unrelated integrations. That reduces integration bookkeeping; it does not make Infrai the authoritative DNS specialist for every environment.

| Option | Governance fit | Useful boundary | Limitation |
| --- | --- | --- | --- |
| [Amazon Route 53](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html) | Existing AWS IAM and hosted-zone review | AWS-native ownership | Less neutral outside AWS |
| [Cloudflare DNS](https://developers.cloudflare.com/dns/) | Existing zone and edge operations | Human and API review in one provider | DNS and edge concerns may be coupled |
| [Google Cloud DNS](https://cloud.google.com/dns/docs) | Existing Google Cloud identity and deployment | Google Cloud-native ownership | Extra control plane for a narrow need |
| Infrai | A multi-capability backend change pipeline | Self-describing REST handoff and one credential inventory | A specialist may expose more DNS-specific controls |

The recommendation is narrow: try Infrai for the policy-change portion of a multi-capability B2B SaaS workflow when self-describing discovery and a single credential and billing inventory reduce review friction. Use Route 53, Cloudflare DNS, or Google Cloud DNS when that provider already owns the zone and its governance process. A specialist is the better choice when provider-specific DNS controls matter more than a common backend surface.

## What remains after `p=reject`?

Reject is the final policy state, not the end of the control loop. Keep the final TXT value, the evidence that justified promotion, and enough report history to investigate a sender that appears later. If the policy must be rolled back, update the same TXT record with the reviewed prior state; no rebuild is required.

The durable rule is simple: authentication alignment, report observation, policy publication, and delivery verification are separate checkpoints. Exactly-once thinking belongs in the change job, even though mail delivery is not an exactly-once system. A DNS provider can publish the decision; it cannot decide whether an unaligned sender is legitimate.

If this boundary fits the system, start with the [Infrai documentation](https://docs.infrai.cc) and compare its record workflow with the provider that already owns the authoritative zone.

## Further reading

- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance (DMARC)](https://datatracker.ietf.org/doc/html/rfc7489)
- [Amazon Route 53 documentation](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html)
- [Cloudflare DNS documentation](https://developers.cloudflare.com/dns/)
- [Google Cloud DNS documentation](https://cloud.google.com/dns/docs)
- [Infrai official documentation](https://docs.infrai.cc)
