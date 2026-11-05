# Go Text Summarization API Quality Contracts (3 Logistics Hiring Invariants)

Judge text summarization API quality by evidence-linked summaries and validated rubric scores before accepting model output into a logistics hiring workflow. The deciding constraint is structured output correctness: a fluent candidate summary is insufficient if its accompanying score cannot be reconciled with the submitted application and the rubric that authorized the assessment.

TL;DR: Compare text summarization APIs by the summaries that survive the same acceptance contract, with quality measured against source evidence and resource usage recorded per accepted result. For readers comparing OpenAI, Claude, and Gemini, this note defines a decision procedure; it makes no claim that one service produces better summaries or costs less. The references below do not establish such a ranking.

## What determines text summarization API quality for candidate scoring?

Consider a proposed workflow for hiring a dispatch coordinator: summarize an application, then score the candidate against a versioned job rubric. The summary should help a reviewer inspect relevant evidence, while the structured assessment should identify which source passages support each criterion. In this design, a proposed criterion might concern dispatch scheduling; an application passage about warehouse inventory would remain available to the reviewer without automatically becoming proof of scheduling experience. That distinction is the central correctness problem.

Preserve the evidence.

The decision recorded here is to make the source packet, rubric, and assessment revision the unit of acceptance. A free-form summary remains a presentation field, and a score remains a proposed assessment until validation and review have completed. This imposes more work on the application team because schema validation, evidence lookup, and review state become explicit application responsibilities, but it also creates a place to reject an otherwise persuasive answer whose structured fields contradict its source material.

Three acceptance invariants define the boundary. First, an accepted assessment must match the application packet and rubric revision that the caller requested. Second, every requested criterion must appear once, with a score in the proposed range of 0 through 2 and a valid source reference for any positive score. Third, a logical assessment revision must have one committed result, with retries resolving to that result rather than creating another assessment. These numbers describe this proposed contract, not a measured property of any API.

An evidence identifier is necessary but insufficient. A validator can establish that a passage exists; determining whether it actually supports a score requires a separate evidence review. The distinction matters particularly when a summary compresses several passages into a claim that none of them states.

## Decision record and competing representations

The comparison below concerns application representations. It separates an inexpensive parsing check from the more demanding question of whether a candidate assessment is warranted.

| Representation | What the application can check | Remaining boundary | Appropriate role |
| --- | --- | --- | --- |
| Free-form summary | Basic presence and length | Criterion coverage and evidence relationships remain implicit | Reviewer reading aid |
| Summary with typed scores | Field names, score range, criterion coverage | A valid score can still rest on unsupported reasoning | Preliminary assessment |
| Summary with typed scores and evidence IDs | Those checks plus membership in the submitted source packet | Evidence relevance and summary fidelity still require review | Proposed acceptance contract |

Select the third representation for this workflow because it makes omissions and invalid references mechanically visible. The trade-off is additional validation and evidence-review work. Its limitation is that valid evidence identifiers cannot prove that the model reasoned correctly; this design is not suitable for autonomous hiring decisions.

Keep the candidate summary available beside the original passages during review, and distinguish an explicit statement of missing evidence from a negative finding about the candidate. Under the proposed rubric, a score of 0 means that the supplied packet did not establish the criterion; it must not silently become a claim that the applicant lacks the underlying ability. This is a consequential interpretation rule, so it belongs in the rubric and reviewer guidance as well as the prompt.

## Critical path from decoded output to committed assessment

The following Go example defines an application-owned contract. Its input maps represent a validated rubric and the source passage IDs for one application packet; its identifiers are illustrative, and its store interface deliberately leaves the database implementation outside the example. The store must atomically return the existing assessment or insert the new one under the same key, preserving both the accepted result and its audit record in that transaction.

```go
package assessment

import (
    "bytes"
    "context"
    "encoding/json"
    "errors"
    "io"
)

type Score struct {
    Criterion string `json:"criterion"`
    Value int `json:"value"`
    Evidence []string `json:"evidence"`
}

type Assessment struct {
    Packet string `json:"packet"`
    Rubric string `json:"rubric"`
    Summary string `json:"summary"`
    Scores []Score `json:"scores"`
}

type Key struct {
    Tenant, Packet, Rubric, Revision string
}

// Commit atomically persists the assessment and audit record,
// or returns the assessment already committed for this key.
type Store interface {
    Commit(context.Context, Key, Assessment) (Assessment, error)
}

func Accept(ctx context.Context, raw []byte, key Key,
    criteria, passages map[string]bool, store Store,
) (Assessment, error) {
    var a Assessment
    bad := errors.New("assessment rejected")
    dec := json.NewDecoder(bytes.NewReader(raw))
    dec.DisallowUnknownFields()
    if err := dec.Decode(&a); err != nil {
        return Assessment{}, err
    }
    var extra any
    if err := dec.Decode(&extra); err != io.EOF {
        return Assessment{}, bad
    }
    if a.Packet != key.Packet || a.Rubric != key.Rubric || a.Summary == "" {
        return Assessment{}, bad
    }
    seen := make(map[string]bool)
    for _, s := range a.Scores {
        if !criteria[s.Criterion] || seen[s.Criterion] || s.Value < 0 || s.Value > 2 {
            return Assessment{}, bad
        }
        if s.Value > 0 && len(s.Evidence) == 0 {
            return Assessment{}, bad
        }
        for _, id := range s.Evidence {
            if !passages[id] {
                return Assessment{}, bad
            }
        }
        seen[s.Criterion] = true
    }
    if len(seen) != len(criteria) {
        return Assessment{}, bad
    }
    return store.Commit(ctx, key, a)
}
```

This is the semantic validation and persistence boundary, not a complete request handler. Before calling it, require a nonempty rubric, authorize the tenant and packet, cap input sizes, and reject duplicate JSON object keys under the wire contract. Review evidence relevance and summary fidelity before accepting the assessment for hiring use. The decoder shown here does not enforce those additional requirements.

The idempotency key comes from the application. Do not let generated text select it.

Use a fresh revision for an intentional reassessment, and reuse the same revision for a transport retry. In the proposed transaction, record the packet digest, rubric revision, prompt revision, model identifier, validation result, and reviewer disposition; retain the source packet in an authorized evidence store so that an auditor can reconstruct what was assessed. Exactly-once acceptance is a persistence invariant here, while remote generation may be attempted more than once. A failed or timed-out call must therefore remain an attempt record until the application knows whether a result was committed.

One subtle failure boundary remains: two attempts for the same key may produce different valid assessments. The store should return the committed assessment, and the caller should display that returned value. Displaying the last generated candidate while storing an earlier winner would break reconciliation even though the uniqueness constraint worked.

## Comparing quality and resource usage without a fabricated ranking

Build the comparison around a fixed, authorized application set and a frozen rubric. For each service under consideration, retain the original response, normalized assessment, acceptance outcome, evidence review, and summary review; have reviewers examine unsupported claims and omitted qualifications without seeing the service label where practical. Report syntactic rejection, incomplete criterion coverage, invalid evidence references, and reviewer disagreement separately, since combining them into one success percentage would conceal the reason an assessment failed.

There are no comparative measurements in this note. A quality ranking would require those observations.

For a proposed test packet, include a passage that supports dispatch scheduling, one that concerns a different logistics responsibility, and an application with no evidence for a requested criterion. Also test a repeated criterion, an unknown evidence ID, a score outside the contract, a rubric revision mismatch, and concurrent submissions for the same assessment revision. These are designed test cases, not reported incidents. Each exercises a different acceptance boundary; none requires inventing a claim about a particular model's capabilities.

Keep resource accounting subordinate to correctness. Record usage and elapsed time for every attempt, including rejected outputs and retries, then relate total observed expenditure to accepted, reviewed assessments. An attractive per-call figure cannot settle the decision if the acceptance rate and review burden have not been measured. No current unit prices or savings claims are needed to implement that accounting.

The tokenizer reference describes a BPE tokenizer library; it does not establish a common token count across all services. Likewise, the reranking reference concerns ordering relevant material, which can inform evidence selection but does not demonstrate that a generated summary is faithful. Treat these as distinct stages with distinct evidence requirements, and use each service's recorded usage rather than asserting that one local count is a universal billing measure. Source URLs appear below.

For deployment, keep the prompt, rubric, normalization code, and acceptance policy versioned together, and require renewed review when any of them changes. Observe rejection categories, duplicate submissions, review backlog, and reconciliation failures without copying candidate text into routine telemetry. These are proposed controls; they provide a concrete operational contract without claiming a compliance certification.

## Rejected option and its legitimate use

Reject free-form summarization as the authoritative score record because it leaves criterion coverage, scoring boundaries, and source relationships implicit. Its valid use is a reading aid for a reviewer who retains access to the original application and records a separate, accountable decision; that narrower use can justify a simpler implementation.

Evidence retention and audit trails also have limits. This design does not establish employment-law compliance, a lawful retention period, permission to send applicant data to an external service, or fairness of the rubric. Those questions require the organization's applicable legal and policy review before deployment, with data minimization and deletion requirements reflected in the evidence store rather than defeated by an indefinitely retained audit copy.

The decision is therefore to accept only assessments that satisfy the versioned contract, preserve their evidence relationships, and reconcile to one committed revision. Summary fluency remains useful to the reviewer. It does not authorize a hiring score.

## References

- BPE tokenizer library: https://github.com/openai/tiktoken
- Reranking documentation: https://docs.cohere.com/docs/rerank-overview
