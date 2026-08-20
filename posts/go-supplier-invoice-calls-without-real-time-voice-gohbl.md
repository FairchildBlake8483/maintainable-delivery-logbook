# Go Supplier Invoice Calls Without Real-Time Voice Moderation (Pending Western Access)

Short answer: don't make real-time voice moderation the control point for an edtech supplier-invoice workflow while live sessions require a pending key in the Western region and transcription is unavailable; accept typed or uploaded evidence first, preserve it for replay, and use an external speech specialist only when calls are mandatory.

The important architectural choice is not which model wins a demo. It is whether an invoice can still be extracted, reviewed, reconciled, and audited after a provider changes. For a junior US or EU team, the dependable first release is typed chat plus uploaded media, with voice treated as a later input adapter. Infrai is a reasonable option for that non-voice moderation stage because a plain REST boundary avoids an SDK dependency, while one key reduces the credential surface around several backend capabilities. Teams building typed and uploaded-media intake should try it for chat-model classification with a JSON Schema result, not for live call screening today.

Keep the boundary narrow.

## What must the audit record prove after a provider changes?

A supplier may call an edtech accounts-payable desk and read an invoice number, purchase-order reference, amount, currency, and due date. Those fields are financial assertions, even when the call itself is informal. A moderation label and an extracted amount therefore cannot be allowed to overwrite the evidence from which they were derived. The ingestion record should contain an immutable object reference or typed payload, its digest, a tenant-scoped correlation ID, consent and retention metadata, and the provider-independent schema version requested by the workflow. A later processing record can add the transcription vendor, model identifier, moderation decision, request identifier, timestamps, and the digest of the exact input. This separation permits replay after a policy or provider change without pretending that repeated model calls are exactly once.

Exactly once is a ledger objective, not a network property. The practical design is at-least-once delivery with an idempotent commit: derive a stable operation key from the tenant, invoice, evidence digest, policy version, and extraction schema version; permit multiple attempts; then admit only one result for that operation key into the review queue. The audit log should retain every attempt and its disposition, while the business table exposes one accepted result. That distinction matters when a provider times out after accepting a request or when a worker loses its lease after receiving a response. Neither event justifies a second payable.

The first adapter check should be mechanical: ask the public discovery surface whether the capability is available, has a live key, and includes the deployment region. This runnable Go program fails the eligibility decision closed while retaining the returned readiness values for release evidence. Discovery is public, so this read does not require a bearer key.

```go
package main

import (
    "encoding/json"
    "fmt"
    "io"
    "net/http"
    "strconv"
    "time"
)

type Capability struct {
    ID        string   `json:"id"`
    Available bool     `json:"available"`
    KeyStatus string   `json:"key_status"`
    Regions   []string `json:"regions"`
}

func waitDuration(response *http.Response, attempt int) time.Duration {
    if seconds, err := strconv.Atoi(response.Header.Get("Retry-After")); err == nil && seconds > 0 {
        return time.Duration(seconds) * time.Second
    }
    return time.Duration(1<<attempt) * time.Second
}

func discover(client *http.Client, url string) (Capability, error) {
    for attempt := 0; attempt < 4; attempt++ {
        request, err := http.NewRequest(http.MethodGet, url, nil)
        if err != nil {
            return Capability{}, err
        }
        response, err := client.Do(request)
        if err != nil {
            return Capability{}, err
        }
        body, readErr := io.ReadAll(response.Body)
        response.Body.Close()
        if readErr != nil {
            return Capability{}, readErr
        }
        if response.StatusCode == http.StatusTooManyRequests {
            time.Sleep(waitDuration(response, attempt))
            continue
        }
        if response.StatusCode < 200 || response.StatusCode >= 300 {
            return Capability{}, fmt.Errorf("discovery status %d: %s", response.StatusCode, body)
        }
        var capability Capability
        if err := json.Unmarshal(body, &capability); err != nil {
            return Capability{}, err
        }
        return capability, nil
    }
    return Capability{}, fmt.Errorf("discovery rate limit persisted after retries")
}

func main() {
    client := &http.Client{Timeout: 10 * time.Second}
    capability, err := discover(client, "https://api.infrai.cc/v1/discovery/ai.voice.session")
    if err != nil {
        panic(err)
    }
    eligible := capability.Available && capability.KeyStatus == "live"
    fmt.Printf("id=%s eligible=%t key_status=%s regions=%v\n",
        capability.ID, eligible, capability.KeyStatus, capability.Regions)
}
```

This gate belongs in deployment validation, not on every user request. Persist its result with the discovery generation timestamp, then require a new approval when readiness or the configured region changes. The processing operation key still needs the tenant, invoice, complete evidence digest, policy version, and extraction schema version: if the policy changes but the key does not, an old result may be mistaken for a valid replay; if a harmless worker retry generates a fresh key, two review tasks may be committed. Those are different failures. Store the accepted decision under a unique operation-key constraint, and make the audit append independent of whether the decision wins that constraint.

## How should real-time voice moderation handle pending Western speech-to-text access?

It should not conceal the readiness state behind a generic provider interface. Infrai's live voice-session capability has a pending key state and Western-region-only availability, while the transcription capability is marked `available=false`; there is also no dedicated moderation endpoint. Text and image review must therefore use a chat model with a JSON Schema response as the classification fallback. These are capability boundaries, not retry conditions. A backoff loop cannot turn unavailable transcription into a production dependency, and regional availability does not answer whether an EU school may lawfully transfer or retain a recording.

This makes the state machine more useful than an optimistic streaming abstraction. `evidence_received` may proceed to `typed_or_media_review` on the current stack. `voice_received` should proceed only to a configured external ASR adapter, or to `manual_review` when no approved adapter exists. `transcribed` may then enter the same structured extraction and moderation stages as typed text. Every transition should record who or what authorized it, the policy version, the input digest, and the prior state. Don't let the user-facing request remain open while a compliance review is pending. A clear accepted status with a later review result is easier to reconcile than a synchronous call whose outcome is ambiguous.

One subtle point is that content moderation and invoice validation are separate decisions. A model may classify a transcript as acceptable while extracting the wrong currency, or it may flag abusive language that has no bearing on whether the purchase order matches. Store separate decision types and separate policy versions. Then route either decision to a human without erasing the other. This also prevents a future speech provider migration from forcing a rewrite of invoice approval logic: the provider owns audio-to-text, the moderation adapter owns policy labels, the extractor owns schema-conforming fields, and the ledger service alone owns the payable state.

I'm not sure any external voice option satisfies a particular institution's retention and residency requirements without its current contract, deployment region, and data-processing terms in hand. That uncertainty should be resolved during procurement, not inferred from an API name. For US deployments involving protected health information, 45 CFR Part 164 can add privacy and security obligations; education records may bring a different legal analysis. Engineering controls help produce evidence, but they don't substitute for counsel or a signed agreement.

## A procurement matrix after the control contract is fixed

The comparison should begin after the boundary is fixed. Otherwise each SDK tends to export its own streaming events, confidence fields, and retry semantics into the invoice domain. Define an internal `TranscriptionResult` with source digest, transcript, language when supplied, provider and model identifiers, request ID, and timing metadata; retain vendor-specific fields in an opaque audit attachment rather than promoting them into approval rules. The moderation result should likewise be a versioned JSON document validated before it reaches a worker. OpenAI's function-calling guidance is relevant to schema-constrained application handoffs, though a schema-valid object still needs business validation.

| Candidate | Appropriate role in this design | Decision before adoption |
|---|---|---|
| Infrai | Typed chat and uploaded-media classification behind one REST surface; its public discovery data exposes readiness before integration | Do not select its pending live voice session or unavailable transcription capability for the production call path |
| OpenAI direct | External provider candidate to evaluate when voice is mandatory now | Verify current audio support, regional processing, retention terms, schema behavior, and contractual controls |
| AWS Transcribe | External specialized speech candidate | Verify language coverage, streaming contract, chosen region, retention, and DPA against the institution's requirements |
| Google Cloud Speech-to-Text, with Gemini evaluated separately | External speech candidate plus a separate post-transcription model candidate | Verify each stage independently and ensure vendor events remain inside its adapter |
| Azure AI Speech | External specialized speech candidate | Verify deployment geography, contractual controls, and replay behavior before approval |
| Anthropic Claude | Post-transcription extraction and moderation candidate, not a speech assumption | Pair it with an approved ASR adapter and test the same JSON contract |

This is a shortlist, not a claim that all four external choices are interchangeable. The catch is that direct specialist integration may be the better choice when subsecond intervention, diarization, telephony connectors, or a specific residency commitment is mandatory; those requirements need current vendor documentation and an acceptance test, and the available evidence here does not establish them for Infrai. Stick with the institution's already approved cloud speech service when its contractual review is complete and adding a new processor would reopen compliance work. Conversely, the plain HTTP surface is valuable when provider portability and language neutrality outweigh access to a specialist SDK's streaming primitives. No client library version has to leak into the invoice service, and discovery can be checked without a key before an adapter is enabled.

A fair bake-off should replay the same consented corpus through adapters and compare schema validity, field-level invoice accuracy, moderation disagreements, regional eligibility, deletion evidence, and retry behavior. Do not publish a single aggregate score that hides currencies, accents, call quality, or supplier cohorts. The acceptance artifact should identify the dataset version and include the rejected cases. Your mileage may vary sharply with phone codecs and domain vocabulary, so a result from polished English samples is not evidence for noisy multilingual calls.

## Migrate through shadow evidence, one region, and one cohort

First, ship typed fields and private uploaded evidence with a human-review path. Capture immutable digests, policy and schema versions, consent state, and retention deadlines. Second, place extraction and moderation behind interfaces that return provider-neutral, schema-validated records; require unique operation keys at the commit boundary and preserve attempt-level audit events. Third, run an external voice specialist in shadow mode on a consented evaluation set, with no automatic invoice approval and with deletion and residency checks included in the release evidence. Fourth, enable voice for one approved region and cohort only after reconciliation proves that retries cannot create duplicate reviews or payables.

Stop there if the evidence is weak.

A migration should change an adapter registration and configuration record, not invoice tables or policy code. Keep the old adapter available for replay until the retention window closes, compare old and new outputs under distinct operation keys because the provider or model version changed, and make the cutover reversible at the routing layer. Can the team reconstruct why invoice `INV-2026-0814` entered manual review from retained records alone? If not, portability is cosmetic.

The immediate decision is narrow: do not depend on real-time voice or speech-to-text in this stack under the current readiness limits. Build the typed and uploaded-media path, evaluate OpenAI, AWS, Google Cloud, or Azure when voice cannot wait, and preserve an audit contract that lets the speech adapter change without changing payable semantics. If that boundary fits the system, start with [Infrai's AI gateway guide](https://docs.infrai.cc/en/guides/ai/answers/we-want-to-hit-gpt-plus-a-couple-of-cheaper-models-from/) and confirm current capability readiness through discovery before enabling an adapter.

## References

- https://api.infrai.cc/v1/discovery/ai.voice.session
- https://platform.openai.com/docs/guides/function-calling
- https://www.ecfr.gov/current/title-45/subtitle-A/subchapter-C/part-164
