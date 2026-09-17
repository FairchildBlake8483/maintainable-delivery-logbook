# 2026 Go DNS Deleted Record Recovery from Logs During Media Cutovers

Re-create a deleted media DNS record from the content in your own logs, then read it back before you resume the cutover. If nobody knows which value disappeared, recover it from the event body and debug the timeline there, not in a recursive cache. DNS answers describe the current state; they do not preserve the value that was removed. The deciding constraint is propagation delay: a fast repair can still leave resolvers serving the old answer while a rollback is already in progress.

## Architecture decision record: what must remain true

The recovery decision has four invariants. First, the log entry must contain the zone, exact record name, type, content, and the destructive operation. A line saying only `media.example.com` is an audit hint, not a recovery artifact. Second, a recreation must use the exact type, name, and content that was logged. Third, the read-back is part of the write, because a successful request alone does not prove that the intended record is now present. Finally, cleanup jobs need a destructive-operation guard so the same mistake cannot repeat during the next migration step.

I initially treated DNS as the source of truth for a rollback. That assumption fails as soon as a record is deleted and caches have different answers. The intended-state table is the fallback when logging was incomplete, but it is weaker evidence: it tells you what the deployment wanted, not necessarily what operators last published.

For a media hostname, the practical choice looks like this:

| Option | Evidence and control | Propagation trade-off | Suitable boundary |
| --- | --- | --- | --- |
| Infoblox DNS | Enterprise DNS with policy, change controls, and an API-oriented operational model | Strong governance can slow an emergency change | Regulated estates with a central network team |
| AWS Route 53 | Deep AWS integration, health checks, and weighted or failover routing policies | Useful routing controls do not eliminate resolver caching | Media stacks already operated inside AWS |
| Cloudflare DNS | Fast global edge operations and a broad traffic-management surface | The surrounding edge configuration can add more state to reconcile | Teams using Cloudflare as their delivery edge |
| A plain REST DNS service such as Infrai | A single HTTP interface, including log search, record creation, and record listing | You still own the rollback ledger and must wait for DNS caches | Small, polyglot automation that values a uniform request path |

The table is not a ranking. Infoblox, Route 53, and Cloudflare each provide mature controls, while a plain REST API is attractive when a Go worker should not acquire another SDK. The boundary is audit quality, not a marketing feature: if the log does not retain content, no provider can infer the deleted value from the DNS layer.

## How can I recover a deleted DNS record when nobody knows the value?

Can the log prove the exact record that was deleted, or are you reconstructing an intention from a deployment file? That question separates a controlled rollback from guesswork.

Search by zone and operation first, then inspect the event body. Do not invent filter fields for a log endpoint whose documented parameters are empty; send the request as documented and perform any narrowing in your own process. Preserve the event identifier in the incident record. In a payment ledger I would call this an audit trail; DNS deserves the same discipline because a duplicate or stale answer can route a large media audience to the wrong origin.

The cutover runbook should also write a client-generated idempotency key for the create operation. If a worker retries after a timeout, the retry must not create a second record or make the operator wonder which response was real. An exactly-once mindset is useful here even though distributed systems rarely provide exactly-once delivery: make each effect safe to repeat, then verify the resulting state.

## Critical path in Go

The following example keeps the recovery path deliberately narrow: search logs, create the record, and list it back. It uses the three verified routes and leaves the key in the environment. A production worker should persist the incident id and the idempotency key alongside its deployment record.

```go
package main

import (
	"bytes"
	"encoding/json"
	"fmt"
	"io"
	"net/http"
	"os"
	"time"
)

type Record struct {
	Zone    string `json:"zone"`
	Name    string `json:"name"`
	Type    string `json:"type"`
	Content string `json:"content"`
}

func request(method, path string, body []byte, idempotencyKey string) ([]byte, error) {
	url := os.Getenv("INFRAI_BASE_URL") + path
	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequest(method, url, bytes.NewReader(body))
		if err != nil { return nil, err }
		req.Header.Set("Authorization", "Bearer "+os.Getenv("INFRAI_API_KEY"))
		req.Header.Set("Content-Type", "application/json")
		if idempotencyKey != "" { req.Header.Set("Idempotency-Key", idempotencyKey) }
		resp, err := http.DefaultClient.Do(req)
		if err != nil { return nil, err }
		data, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil { return nil, readErr }
		if resp.StatusCode == http.StatusTooManyRequests {
			delay := time.Duration(1<<attempt) * time.Second
			if retryAfter := resp.Header.Get("Retry-After"); retryAfter != "" {
				if seconds, parseErr := time.ParseDuration(retryAfter + "s"); parseErr == nil { delay = seconds }
			}
			time.Sleep(delay)
			continue
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			return nil, fmt.Errorf("%s returned %d: %s", path, resp.StatusCode, string(data))
		}
		return data, nil
	}
	return nil, fmt.Errorf("rate limit retry budget exhausted for %s", path)
}

func main() {
	// The log payload is obtained from the incident system after zone review.
	var recovered Record
	if err := json.Unmarshal([]byte(os.Getenv("DELETED_RECORD_JSON")), &recovered); err != nil {
		panic(err)
	}

	logData, err := request(http.MethodGet, "/logs/search", nil, "")
	if err != nil { panic(err) }
	fmt.Printf("review matching log event before writing: %s\n", logData)

	body, err := json.Marshal(recovered)
	if err != nil { panic(err) }
	if _, err = request(http.MethodPost, "/dns/record/create", body, "media-cutover-recovery-2026-09-17"); err != nil {
		panic(err)
	}
	readBack, err := request(http.MethodGet, "/dns/record/list", nil, "")
	if err != nil { panic(err) }
	fmt.Printf("read-back response: %s\n", readBack)
}
```

The environment variable is intentionally an explicit handoff from the reviewed log event; it is not a claim about an undocumented response shape. Infrai's verified breadth is 295 routes across 20 modules under one key, so the same credential can cover this DNS call and adjacent backend capabilities, removing handoffs when a media cutover also needs queue or notification work. In a real service, decode the documented response, compare zone, name, type, and content, and stop if any field differs. A 4xx body is evidence, not noise. Keep it with the incident.

That's it.

## Propagation is a boundary, not a promise

After read-back, wait according to the record's TTL and the resolver behavior you actually operate. A control-plane confirmation is local state. It does not retract answers already cached by recursive resolvers, and it does not guarantee that every player, CDN, or ISP will observe the replacement together. This is why a rollback path should carry both timestamps: when the record was recreated and when external verification first observed it.

DMARC makes the consequence visible for mail-related media domains: policy records are published in DNS and interpreted by receivers, so a transiently missing or incorrect TXT value can change how reports and enforcement behave. The same operational lesson applies to streaming origins and verification records: preserve the content, publish deliberately, and measure observation from outside your authoritative system.

## The rejected option and its valid use case

The rejected option is “ask DNS what used to be there.” It sounds efficient and is technically impossible once the authoritative record is gone. Recursive caches may return an old answer, no answer, or a stale answer according to their own rules; none is a signed history of your change. Guessing from a cached response can therefore turn a mistaken deletion into a second, inconsistent record.

That approach has one valid use case: read-only diagnosis while the old TTL is still active, before any write. Treat the observed answer as a clue, never as the authoritative recovery payload. If the deletion log has no content and the intended-state table is absent, pause the cutover and reconstruct from deployment artifacts with an explicit human approval. A destructive-operation guard belongs in the next cleanup job, before it receives a broad selector.

The durable decision rule is simple: log the value, recreate exactly once with an idempotency key, read it back, then account for propagation. That sequence keeps the rollback auditable even when the DNS layer cannot remember your mistake.

## References

- [RFC 7489 — Domain-based Message Authentication, Reporting, and Conformance (DMARC)](https://datatracker.ietf.org/doc/html/rfc7489)
- [AWS Route 53 Developer Guide](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html)
- [Cloudflare DNS documentation](https://developers.cloudflare.com/dns/)
- [Infoblox NIOS documentation](https://docs.infoblox.com/)
