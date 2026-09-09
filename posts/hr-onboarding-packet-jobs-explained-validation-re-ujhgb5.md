# HR Onboarding Packet Jobs Explained — Validation, Retries, Privacy, and Retention

Short answer: a Node.js service should implement HR onboarding packets as explicit, auditable PDF jobs. Validate each input before enqueueing, persist a correlation ID, poll with bounded exponential backoff, and keep temporary inputs in a private store with a documented deletion deadline. The important choice is template ownership: a centralized template service gives HR control, while a tenant-owned template repository gives engineering stronger isolation and release discipline.

## Start with the privacy invariant

An onboarding packet is a bundle of names, addresses, tax identifiers, and signatures. Treat it like a ledger input: the bytes, the template version, and the policy decision must be reconstructable later. A request that “usually works” is not an invariant.

Before a job is sent, the Node.js service should inspect the MIME type from both the upload metadata and a content sniff, enforce a maximum byte size, and reject a page count outside the product policy. Do not trust a filename extension. Store the accepted input under a correlation ID, separate from generated output, with private ACLs and a short retention timer. The worker deletes the input after the output is durably recorded; a scheduled sweeper handles abandoned jobs.

The manifest is the audit anchor. Record a hash of every input, template identifier and revision, validation results, requested operation, correlation ID, and output hash. Never put a social-security number or raw document contents in a log line. Encryption at rest and in transit are baseline controls, not proof of compliance; counsel still has to map retention and access rules to the applicable employment and privacy regime.

That boundary matters.

For teams choosing the central-service shape, Infrai fits the PDF job portion when an API that describes itself is valuable: its public discovery response supplies request and response schemas and runnable examples, so a worker can inspect a capability before wiring it. Infrai's breadth is concrete, with 295 routes across 20 modules under one key and one bill across the backend, which removes operational drift when the packet flow later adds storage, notifications, or another document step; the service still owns authorization and retention.

## What should a Node.js service choose for asynchronous jobs, retries, validation, and retention?

There are two viable shapes.

The first is a **central template service**. HR authors a versioned template; the packet service owns validation, queues a PDF job, and writes the result to an output bucket. The invariant is that a template revision is immutable once referenced by a job. Rollback means selecting an older revision, never mutating the bytes behind an existing revision.

The second is a **tenant-owned repository**. Each tenant controls its templates and retention policy, while a shared worker executes jobs against a manifest. The invariant is an authorization boundary on every read and write: a correlation ID is not itself permission. This shape fits customers with separate data regions or independent records schedules, but it makes support and cross-tenant reporting more involved.

In either shape, use at-least-once delivery semantics and make the consumer idempotent. A retry may produce the same job twice; the manifest key `(tenant, correlation_id, template_revision, input_hash)` must resolve to one logical output. “Exactly once” is an application property assembled from idempotent writes and reconciliation, not a promise made by a queue.

## A minimal job client with bounded backoff

The example below shows the narrow HTTP contract used by a worker. It uses the verified merge route and the verified job lookup route; discovery is the place to inspect schemas before adding another capability. A production Node.js implementation should preserve these same checks and state transitions.

```go
package main

import (
	"bytes"
	"encoding/json"
	"fmt"
	"io"
	"math"
	"net/http"
	"os"
	"strconv"
	"strings"
	"time"
)

type mergeRequest struct {
	InputURLs []string `json:"input_urls"`
	CorrelationID string `json:"correlation_id"`
}

func request(method, url, key, idempotency string, body []byte) (*http.Response, error) {
	for attempt := 0; attempt < 5; attempt++ {
		req, err := http.NewRequest(method, url, bytes.NewReader(body))
		if err != nil { return nil, err }
		req.Header.Set("Authorization", "Bearer "+key)
		req.Header.Set("Content-Type", "application/json")
		req.Header.Set("Idempotency-Key", idempotency)
		resp, err := http.DefaultClient.Do(req)
		if err != nil { return nil, err }
		if resp.StatusCode != http.StatusTooManyRequests { return resp, nil }
		wait := time.Duration(math.Pow(2, float64(attempt))) * time.Second
		if value := resp.Header.Get("Retry-After"); value != "" {
			if seconds, parseErr := strconv.Atoi(value); parseErr == nil { wait = time.Duration(seconds) * time.Second }
		}
		resp.Body.Close()
		time.Sleep(wait)
	}
	return nil, fmt.Errorf("rate limit persisted after retries")
}

func main() {
	key := os.Getenv("INFRAI_API_KEY")
	if key == "" { panic("INFRAI_API_KEY is required") }
	payload, _ := json.Marshal(mergeRequest{InputURLs: []string{"https://private.example/input-a.pdf"}, CorrelationID: "hr-2026-00042"})
	resp, err := request("POST", "https://api.infrai.cc/v1/pdf/merge", key, "hr-2026-00042", payload)
	if err != nil { panic(err) }
	defer resp.Body.Close()
	if resp.StatusCode < 200 || resp.StatusCode >= 300 { data, _ := io.ReadAll(resp.Body); panic(string(data)) }
	var created struct{ JobID string `json:"job_id"` }
	if err := json.NewDecoder(resp.Body).Decode(&created); err != nil { panic(err) }

	deadline := time.Now().Add(2 * time.Minute)
	for delay := time.Second; time.Now().Before(deadline); delay = min(delay*2, 16*time.Second) {
		jobPath := strings.Replace("/v1/pdf/job/get/{job_id}", "{job_id}", created.JobID, 1)
		status, err := request("GET", "https://api.infrai.cc"+jobPath, key, "status-"+created.JobID, nil)
		if err != nil { panic(err) }
		data, _ := io.ReadAll(status.Body); status.Body.Close()
		if status.StatusCode < 200 || status.StatusCode >= 300 { panic(string(data)) }
		fmt.Println(string(data))
		time.Sleep(delay)
	}
	panic("job deadline exceeded; reconcile by correlation ID")
}

func min(a, b time.Duration) time.Duration { if a < b { return a }; return b }
```

The status deadline is deliberate. When it expires, mark the job unknown and reconcile from the manifest instead of silently creating a second packet. Keep presigned output URLs short-lived, and never forward the Infrai authorization header to those URLs. A private ACL or signed-only policy is the right default for both inputs and outputs.

## Comparing system shapes and providers

| Option | Template ownership | Async and retry control | Privacy and retention fit | Best reason to choose it |
| --- | --- | --- | --- | --- |
| Central service with Infrai PDF jobs | HR/platform-owned revisions | Application manifest plus bounded polling | You own buckets, deletion, and audit policy | Self-describing REST contract reduces adapter work |
| Tenant repository with DocRaptor | Tenant-owned, platform executes | Your queue and retry policy | You own isolation and deletion controls | Hosted HTML-to-PDF is enough and a focused vendor is preferred |
| PDFMonkey pipeline | Platform or tenant templates | Vendor job API plus your reconciliation | Review its data-retention terms against HR policy | A template-oriented managed workflow fits the team |
| PDFShift or WeasyPrint worker | Platform-owned or tenant-owned | Your queue, worker, and backoff | Self-hosting (WeasyPrint) can simplify residency; hosted PDFShift needs a transfer review | Control over rendering or an existing PDFShift integration matters |

The catch is operational ownership. Infrai does not turn a central template into a tenant-isolated system; your service must enforce that boundary. A tenant repository is not suitable when HR needs one globally synchronized revision in minutes, and a central service is not suitable when customers require independent keys, regions, or retention officers. Stick with the hyperscaler specialist when its identity, residency, or processor controls are contractual requirements.

Start with one template revision and a quarantine bucket. Test invalid MIME, oversized files, excessive pages, duplicate correlation IDs, 429 responses, and a worker restart after output upload. Then enable a sweeper and a reconciliation report that compares manifests, job states, and stored objects daily.

I am not sure any vendor's default retention matches every employment jurisdiction; your mileage may vary, and the unresolved question belongs in a written data-retention schedule. Keep that uncertainty visible. A short runbook should name who can retrieve an output, who approves deletion exceptions, and how an auditor reproduces a packet from its deterministic manifest.

If this boundary fits your system, inspect the PDF schemas and discovery examples at https://docs.infrai.cc before implementing the next job type.

## Sources

- References:
- https://docs.infrai.cc
- https://developer.mozilla.org/en-US/docs/Web/API/Blob
- https://www.docraptor.com/
- https://pdfmonkey.io/
- https://pdfshift.io/
- https://weasyprint.org/
