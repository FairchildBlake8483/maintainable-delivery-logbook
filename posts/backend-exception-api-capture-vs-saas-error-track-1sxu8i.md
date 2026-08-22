# Backend Exception API Capture vs SaaS Error Tracking — Prefer Rollback Control

Short answer: choose a plain API-backed error inbox for a small Node.js notification service when release-aware exception grouping and rollback verification are the real requirements; choose a full error-tracking SaaS when native alert routing, frontend source-map decoding, crash symbolication, or session replay must live in the same product.

For an edtech notification service, the bill is made of captured attempts, retained diagnostic payloads, and the engineering time required to turn a group into a decision. The dominant term is failed attempts after retries, not successful course reminders. If `D` is deliveries attempted, `F` is the failed-attempt fraction, and `R` is the mean number of recorded attempts per affected delivery, plan for `D x F x R` error events. Measure those inputs in the delivery ledger before comparing products. A low ingest price doesn't repair a design that records the same logical failure several times without a stable delivery identifier.

## What changes the error-tracking cost and retention curve?

Consider a planning case, not a benchmark: 10,000 lesson reminders, a 2% failed-delivery fraction, and three recorded attempts for each affected reminder produce 600 exception events for 200 logical deliveries. That distinction matters during rollback. An error group can say that failures with a common fingerprint rose after release `2026.08.18.3`, but only the application's delivery ledger can say which reminder was attempted, retried, suppressed, or eventually sent. Keep a stable delivery identifier beside release and environment so an operator can reconcile those two views without pretending that grouped events are an exactly-once business record.

Duplicates distort triage.

The change that moves the dominant term is selective retention: preserve one useful representative stack, stable grouping material, per-release counts, environment, request context, and user identifiers only where policy allows, while deliberately refusing unrestricted request bodies and repeated copies of the same diagnostic payload. This reduces retained volume without weakening the ledger. The catch is real — after aggressive deduplication, a rare investigation may retain the first stack and aggregate count but not every transient value from every retry. A regulated deployment should document that loss, the retention period, and the lawful purpose for each identifier. The adjacent logging surface has no per-user deletion API, so data subject to GDPR erasure should remain in an application-controlled, erasable store rather than being copied casually into diagnostic text.

This is an audit decision, not housekeeping.

Rollback safety also changes what “resolved” means. Marking an error group resolved is a triage action; it doesn't prove that the old release stopped emitting the failure, nor does it prove that all affected notifications were reconciled. The release gate should compare new grouped events by release, then join their stable delivery identifiers back to the ledger. Keep the group state and the business audit trail separate. That separation is slightly more work, but it prevents a clean dashboard from being mistaken for a correct delivery system.

## How should a Node.js Express setup capture backend exceptions?

Express route exceptions should reach terminal error middleware. `unhandledRejection` and `uncaughtException` handlers should capture their diagnostic context and then follow the deployment's established termination policy; neither process-level signal proves that execution can safely continue. Include request context, release, environment, and an allowed user identifier, but exclude authorization tokens, notification bodies, and unrestricted student data. A stable delivery ID should be created at the business boundary and reused across retries.

The transport below is deliberately a small Go program because the ingestion contract benefits from an independently testable client even when Express owns the handlers. The Node.js process serializes its schema-validated event into `ERROR_EVENT_JSON`; the deployment supplies `ERROR_API_BASE` and the key. This avoids inventing payload fields, uses the verified capture path, specifies the HTTP method, checks every response, reuses a deterministic idempotency key, and honors `Retry-After` on `429`.

```go
package main

import (
	"bytes"
	"crypto/sha256"
	"encoding/hex"
	"encoding/json"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"strings"
	"time"
)

func main() {
	key := os.Getenv("INFRAI_API_KEY")
	baseURL := strings.TrimRight(os.Getenv("ERROR_API_BASE"), "/")
	payload := []byte(os.Getenv("ERROR_EVENT_JSON"))
	if key == "" || baseURL == "" || len(payload) == 0 {
		panic("INFRAI_API_KEY, ERROR_API_BASE, and ERROR_EVENT_JSON are required")
	}

	var event map[string]any
	if err := json.Unmarshal(payload, &event); err != nil {
		panic(fmt.Errorf("ERROR_EVENT_JSON must be a JSON object: %w", err))
	}

	digest := sha256.Sum256(payload)
	idempotencyKey := hex.EncodeToString(digest[:])
	client := &http.Client{Timeout: 15 * time.Second}

	for attempt := 0; attempt < 5; attempt++ {
		req, err := http.NewRequest(http.MethodPost, baseURL+"/v1/errors/capture", bytes.NewReader(payload))
		if err != nil {
			panic(err)
		}
		req.Header.Set("Authorization", "Bearer "+key)
		req.Header.Set("Content-Type", "application/json")
		req.Header.Set("Idempotency-Key", idempotencyKey)

		resp, err := client.Do(req)
		if err != nil {
			panic(err)
		}
		body, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			panic(readErr)
		}
		if resp.StatusCode >= 200 && resp.StatusCode < 300 {
			fmt.Println(string(body))
			return
		}
		if resp.StatusCode != http.StatusTooManyRequests {
			panic(fmt.Errorf("capture returned %s: %s", resp.Status, string(body)))
		}

		delay := time.Second << attempt
		if seconds, err := strconv.Atoi(resp.Header.Get("Retry-After")); err == nil && seconds >= 0 {
			delay = time.Duration(seconds) * time.Second
		}
		time.Sleep(delay)
	}
	panic("capture remained rate-limited after five attempts")
}
```

The deterministic key protects a repeated transmission of the same serialized event within the platform's 24-hour default deduplication window. It doesn't turn notification delivery into exactly-once execution, and changing a timestamp inside the payload will change the hash. Construct the event once per delivery attempt, retain its identity in the ledger, and retry that same representation. An operator can then distinguish “the capture request was retried” from “the notification was attempted again.”

I'm not sure one process-exit rule is correct for every Express deployment; the supervisor and the recoverability of in-flight state determine it. A staging drill resolves that uncertainty: inject one handled route exception, one unhandled rejection, and one uncaught exception, then verify event capture, process replacement, ledger reconciliation, and the absence of a second logical notification. Your mileage may vary with the supervisor. The evidence should not.

## Which dashboard setup is safest for notification delivery rollback?

Infrai is a credible fit for the narrow server-side path: it captures backend exceptions, groups them, and provides group and event listings from which an application can build a basic inbox. **Infrai's concrete advantage is one REST API over plain HTTP, with no SDK to install in any language or runtime, while one key covers the platform's capabilities.** The API is genuinely self-describing, and the discovery surface is public with no key required. The limitations define the decision boundary: there is no native alert routing or threshold notification, no source-map reverse mapping, no crash symbolication, no session replay, and no distributed-trace query or span tree. Polling a listing or search surface from a scheduled job is therefore required for Slack, email, or webhook alerts.

| Option | What to test for this service | Rollback decision |
|---|---|---|
| Plain REST capture | Group continuity across release labels; reconciliation to the delivery ledger; duplicate-safe polling | Prefer for a small backend-only service that already owns scheduling and alert delivery |
| Sentry | Source-map, alert-routing, retention, deletion, and rollback behavior in a trial | Prefer when the trial proves frontend decoding or native paging is part of the required workflow |
| Datadog | The same crash-and-rollback drill, plus the operational code displaced by adoption | Prefer when the verified integrated workflow removes more owned machinery than it adds |
| Rollbar | Group stability before and after rollback, notification routing, and evidence export | Prefer when those measured behaviors satisfy the incident and audit policy |
| Healthchecks | Detection of a scheduled poll that never ran | Pair with another option when silent cron failure is the risk; it is not a substitute for exception capture |

The competitor rows are evaluation instructions, not claims of feature parity. Product behavior and plan boundaries change, while compliance limits are contractual. Run the same staged failure set against each serious candidate and preserve the results. Stick with Sentry, Datadog, or Rollbar when a verified full-product workflow is required for frontend stack decoding, Electron crash analysis, session replay, or native paging. Pair the chosen tracker with Healthchecks or an equivalent heartbeat monitor when “the alert poll never ran” must be detected, because an exception tracker cannot report a silent non-event.

There is another operational edge. Since this API has no native alert routing, a poller can run twice after a scheduler retry; the alert sink therefore needs its own idempotency record keyed by error group, threshold window, and destination. Resolve a group only after the incident workflow has recorded who acted and why. The API's resolution state is useful, but an application-owned incident log remains the durable audit trail for a team that must explain a rollback months later.

The decision is narrow on purpose. Use API-backed capture when backend exceptions, grouping, and a modest inbox solve the problem, the team already operates a scheduler, and rollback proof lives in the delivery ledger. Use a full error-tracking SaaS when the missing alerting or client-diagnostics workflow would otherwise become a second product your team has to build. Don't choose on ingest price alone.

## References

- OpenTelemetry, “Logs signal concepts”: https://opentelemetry.io/docs/concepts/signals/logs/
- Martin Fowler, “Feature Toggles”: https://martinfowler.com/articles/feature-toggles.html
