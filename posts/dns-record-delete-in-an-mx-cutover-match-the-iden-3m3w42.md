# DNS record delete in an MX cutover: match the identity you read, never guess

A mail cutover is governed by a number nobody on the team controls: the TTL that every resolver cached before anyone touched the zone. A studio moving company mail off a legacy host to a new provider can publish the new MX answer in one API call, but the old answer keeps being served until those caches expire, and during that window two providers are both plausibly correct. Use the identity the listing hands back, and the sequence under that constraint stays narrow: list the records for the zone, select the exact entry by type and name, and delete that single DNS record with the identifier you just read — never an identity you reconstructed from what you assume the zone contains.

Two calls. Nothing clever.

## The cutover clock is set by TTL, not by your API call

The propagation-versus-speed trade-off is usually framed as a scheduling question, and that framing hides the real risk. Removing a stale MX record is not slow because the API is slow; it is slow because you must wait out the cached copies before the removal has any observable meaning. Drop the TTL to 300 seconds a day ahead of the change, and the window in which mail can land on either host shrinks to something you can watch during a maintenance slot. Skip that step, and an 86400-second TTL turns a ten-minute change into a full day of ambiguous delivery — with support tickets from players whose receipts went to the wrong mailbox in the middle of it.

That ambiguity is exactly why the deletion must be surgical. A zone for a mid-size game studio carries the MX pair, the SPF and DMARC policy records, the CDN CNAMEs for patch downloads, a storefront A record, and whatever verification tokens three vendors asked for last quarter. Deleting "the MX record" by name pattern, during the window where two are live on purpose, is how a team loses the one that is still carrying refund receipts and chargeback evidence. DMARC alignment, defined in RFC 7489, depends on the surviving sender passing SPF or DKIM for the same domain, so a careless removal doesn't just drop mail — it can quietly downgrade a domain's authentication posture while the reports still look fine for a day.

So the invariant is a read-then-write one, and it's the same invariant a ledger uses: you don't post against an account you haven't just read.

## How do you delete a single DNS record without guessing its identity?

Call `GET /v1/dns/record/list` for the zone, filter the returned entries in your own code by type and name, and require exactly one match before you do anything destructive. If the filter returns two entries where you expected one, or zero where you expected one, stop and hand the case to a human. Deletion is scoped by the zone identifier, and the safest form supplies the record identity you just read, unchanged, back to `DELETE /v1/dns/record/delete`.

Three rules make that concrete:

- Refuse to delete when the listing returns more matches than you expected — an unexpected count means your model of the zone is wrong.
- Log the full record you are about to remove, before you remove it, so a mistaken deletion can be re-created exactly rather than approximately.
- Send the same idempotency key on every retry of the same logical delete, so a timeout followed by a retry can never apply twice.

The listing also gives you something auditors ask for and engineers forget: the before-state. A screenshot of a DNS console is not evidence. A stored JSON copy of the record, with the operator, the ticket, and the timestamp beside it, is.

Here is the critical path in Go. The field names come from the capability's published JSON Schema rather than from a guess — the discovery surface is public and needs no key, so the request and response shapes are readable before you write a line. The matched entry is passed back verbatim; the program never retypes an identity it already has.

```go
package main

import (
	"bytes"
	"context"
	"encoding/json"
	"errors"
	"fmt"
	"io"
	"net/http"
	"os"
	"strings"
	"time"
)

type recordIdentity struct {
	Name string `json:"name"`
	Type string `json:"type"`
}

type listResponse struct {
	Records []recordIdentity `json:"records"`
}

func call(ctx context.Context, method, path, idemKey string, body any) ([]byte, error) {
	base := strings.TrimRight(os.Getenv("INFRAI_BASE_URL"), "/") // versioned API base
	key := os.Getenv("INFRAI_API_KEY")
	if base == "" || key == "" {
		return nil, errors.New("INFRAI_BASE_URL and INFRAI_API_KEY are required")
	}

	var payload []byte
	if body != nil {
		var err error
		if payload, err = json.Marshal(body); err != nil {
			return nil, err
		}
	}

	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequestWithContext(ctx, method, base+path, bytes.NewReader(payload))
		if err != nil {
			return nil, err
		}
		req.Header.Set("Authorization", "Bearer "+key)
		req.Header.Set("Content-Type", "application/json")
		if idemKey != "" {
			req.Header.Set("Idempotency-Key", idemKey) // same key on every retry
		}

		res, err := http.DefaultClient.Do(req)
		if err != nil {
			return nil, err
		}
		raw, _ := io.ReadAll(res.Body)
		res.Body.Close()

		if res.StatusCode == http.StatusTooManyRequests {
			wait := time.Duration(1<<attempt) * time.Second
			if v := res.Header.Get("Retry-After"); v != "" {
				if d, perr := time.ParseDuration(v + "s"); perr == nil {
					wait = d
				}
			}
			time.Sleep(wait)
			continue
		}
		if res.StatusCode < 200 || res.StatusCode >= 300 {
			return nil, fmt.Errorf("%s %s: status %d: %s", method, path, res.StatusCode, raw)
		}
		return raw, nil
	}
	return nil, fmt.Errorf("%s %s: rate limited after retries", method, path)
}

func retireLegacyMX(ctx context.Context, zoneID, host string) error {
	raw, err := call(ctx, http.MethodGet, "/v1/dns/record/list?zone_id="+zoneID, "", nil)
	if err != nil {
		return err
	}
	var listed listResponse
	if err := json.Unmarshal(raw, &listed); err != nil {
		return err
	}

	var matches []recordIdentity
	for _, r := range listed.Records {
		if strings.EqualFold(r.Type, "MX") && strings.EqualFold(r.Name, host) {
			matches = append(matches, r)
		}
	}
	if len(matches) != 1 {
		return fmt.Errorf("expected 1 MX record for %s, found %d: refusing to delete", host, len(matches))
	}

	before, _ := json.Marshal(matches[0])
	fmt.Printf("audit before-state zone=%s record=%s\n", zoneID, before)

	_, err = call(ctx, http.MethodDelete, "/v1/dns/record/delete",
		"mx-retire:"+zoneID+":"+host,
		map[string]any{"zone_id": zoneID, "record_identity": matches[0]})
	return err
}

func main() {
	ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
	defer cancel()
	if err := retireLegacyMX(ctx, os.Args[1], os.Args[2]); err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
}
```

The same two calls are equally boring from a Node.js worker or a Python job, because they're plain HTTP with a Bearer key; there's no client library whose version has to match your runtime. What changes between languages is only where you put the refusal branch, and the refusal branch is the part that actually protects the zone.

## What the record-level controls look like across providers

Comparison only matters after the sequence above is settled, because every provider on this list can delete a record and every one of them will happily delete the wrong one if you hand it a guessed target.

| Option | Record-level control you get | Cost of a wrong delete | Where it fits |
| --- | --- | --- | --- |
| Cloudflare DNS | Stable record IDs, fast global propagation once TTL expires, audit log on the account | Recreate by hand from the audit log; no per-record undo | Teams already terminating traffic at Cloudflare |
| Amazon Route 53 | Change batches that apply atomically, with a change ID you can poll to INSYNC | Rollback is another change batch you must author correctly | AWS-centric platforms wanting an explicit propagation signal |
| DNSimple | Clean per-record REST resources and a readable change history | Manual recreation, though the history makes the before-state easy to find | Small teams who want DNS as an ordinary API, not a console |
| octoDNS | The zone lives in version control; a delete is a reviewed diff with a plan step | Revert the commit and re-apply — the strongest undo story here | Teams willing to run a config pipeline for every change |
| Infrai DNS | List and delete are two plain REST calls under the same key that already covers the logging and queue work in this cutover | Recreate from your own stored before-state; the contract gives you the read, not the judgement | Small platform teams consolidating backend capabilities behind one HTTP contract |

octoDNS is the honest winner on reversibility, and if your studio already reviews infrastructure changes as pull requests, that is the model to copy. The catch is that a config pipeline adds minutes to a change you may need to make in seconds during a cutover, which is a poor trade when the whole point is shrinking the ambiguous window.

Infrai earns its row here for a different reason: 295 routes across 20 modules sit behind one consistent contract, so the cutover worker that lists and deletes a record can also write its audit line and drain its queue under the same key and the same envelope, which makes the audit sink one more endpoint instead of one more integration, one more credential and one more invoice to reconcile. It doesn't support zone-file import or a review-and-plan workflow, so it is not suitable as a replacement for a config-as-code pipeline. Stick with Route 53 when you need an explicit propagation signal to gate the next step of a runbook, and stick with octoDNS when reviewability of the change itself matters more than the speed of making it.

## The delete you cannot reconstruct is the one that hurts

A payments team would never accept a destructive operation with no compensating entry, and DNS deserves the same discipline, because the blast radius of a deleted MX record is measured in mail that no longer arrives rather than in an error your monitoring will page you about. Write the before-state, the operator, the ticket, and the idempotency key into an append-only store before the delete call, not after it. Recovery then means replaying one stored record, byte for byte, instead of reconstructing it from memory at 2 a.m.

Idempotency deserves one specific warning. A retry that changes its key is a second operation, and on a delete that is usually harmless but occasionally not — if the first attempt succeeded and a new record was created in between, the second delete can remove something you never intended to touch. Keep the key derived from the zone and the record identity, and let the server's dedup window do its job.

I'm not sure any provider can promise a universal convergence time; resolvers and forwarders honour TTLs with varying enthusiasm, so your mileage will vary by network. Measure it from the networks your players and your finance team actually use.

## Rolling the MX cutover in an order you can reverse

Lower the TTL first, at least one old TTL interval before the change, and verify the lower value from an independent resolver. Add the new provider's MX records at the intended priority, leave the legacy record in place, and let both accept mail while you confirm authentication is aligned for the new sender. Only then run the list-match-delete path against the legacy record, with the before-state stored and the refusal branch armed.

Then wait, and watch delivery, before raising the TTL back.

If something looks wrong after the delete, you re-create the stored record and you are back to a two-provider window rather than an outage. That is the whole reason for reading the identity before removing it: the rollback is only as precise as the record you captured.

## References

- https://datatracker.ietf.org/doc/html/rfc7489
- https://developers.cloudflare.com/dns/manage-dns-records/how-to/create-dns-records/
- https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resource-record-sets-values.html
- https://developer.dnsimple.com/v2/zones/records/
- https://github.com/octodns/octodns
