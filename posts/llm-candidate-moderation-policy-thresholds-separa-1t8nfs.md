# LLM Candidate Moderation: Policy Thresholds Separating False Positives from Review Queues

Short answer: route candidate-generated content through separate allow, review, and block decisions, then calibrate those decisions against labeled examples for each policy and jurisdiction; never let a moderation score silently become a hiring score.

For an edtech system that scores candidates against a job rubric, structured output correctness is the first constraint. A moderation model may emit a perfectly valid probability vector and still produce the wrong operational result. A quoted slur in an anti-harassment training answer, a security candidate's explanation of an exploit, or a medical example can resemble prohibited content after context has been compressed. The false positive happens at the policy boundary: the classifier detects a pattern, while the product treats detection as proof that blocking is appropriate.

This distinction matters because the downstream action can affect access to an assessment and, indirectly, an employment opportunity. The safe design is deliberately conservative about automation — but conservative doesn't mean “block more.” It means preserving evidence, isolating uncertain cases, and making every consequential transition reproducible.

## Why do LLM moderation false positives require policy thresholds for US and EU user content?

The model and the policy answer different questions. A model estimates whether text resembles a category represented in its training and evaluation data. Policy decides what the platform should do with that estimate, given the speaker, the quoted material, the assessment task, the jurisdiction, and the harm of a mistaken action. One global cutoff collapses all of those dimensions into a number that cannot explain itself.

The actions are not interchangeable:

| Action | Required evidence | Operational consequence |
| --- | --- | --- |
| Allow | Below the approved review band | Continue to rubric scoring |
| Review | Ambiguous or context-dependent match | Pause safety disposition, preserve assessment state |
| Block | Approved automatic-enforcement band | Stop publication or access, retain an appeal path |

Consider a candidate answering a rubric item about incident response. The submission contains an example phishing message, then explains the indicators that reveal it. A content classifier can reasonably assign elevated scores to fraud or credential theft. Yet blocking the answer erases the surrounding educational purpose; allowing every such answer without inspection would be equally careless. The correct middle state is review, with enough context for a reviewer to distinguish instruction from endorsement. This is why policy thresholds belong in a versioned decision layer rather than inside an opaque model call.

The US/EU distinction should also be configuration, not scattered conditional code. Legal and compliance teams may require different notices, retention periods, escalation paths, or human-review controls. Those requirements can change, and I'm not sure a single regional label will ever capture every applicable obligation: establishment, candidate location, customer contract, and the nature of the decision can all matter. Resolve that uncertainty through counsel-approved policy versions and recorded jurisdiction inputs, rather than an engineer's guess at request time.

Keep two ledgers. The moderation ledger records content-safety evidence and action. The assessment ledger records rubric evidence and score. They may share a submission identifier, but a moderation category must not add or subtract rubric points; a reviewed submission should return to the same scoring path after clearance. That separation makes reconciliation possible when a policy changes or an appeal succeeds.

## Treat the model output as evidence, not a verdict

A useful contract has three layers: typed model evidence, deterministic policy evaluation, and an append-only decision record. Parse model output under a strict schema. Reject missing categories, non-finite scores, unknown schema versions, and incomplete provenance before applying policy. Don't coerce malformed output into zeroes; that turns an integration failure into an apparently safe decision.

The policy engine then maps evidence to `allow`, `review`, or `block`. `Block` should be reserved for categories and confidence bands that policy owners have explicitly approved for automatic enforcement. The review band absorbs ambiguity. It is not a softer block: it needs a service-level objective, queue ownership, reviewer guidance, and an appeal route. If the queue cannot meet its service objective, reduce automated scope or add reviewer capacity rather than quietly treating a timeout as rejection.

The following Go example keeps the mechanism intentionally small. The thresholds are illustrative configuration values, not universal safety constants; production values must come from a labeled corpus that represents the platform's candidate submissions.

```go
package moderation

import (
	"errors"
	"fmt"
	"math"
	"sort"
)

type Action string

const (
	Allow  Action = "allow"
	Review Action = "review"
	Block  Action = "block"
)

type Evidence struct {
	SchemaVersion string             `json:"schema_version"`
	ModelVersion  string             `json:"model_version"`
	Scores        map[string]float64 `json:"scores"`
}

type Band struct {
	ReviewAt float64
	BlockAt  float64
}

type Decision struct {
	SubmissionID string   `json:"submission_id"`
	PolicyVersion string  `json:"policy_version"`
	Jurisdiction string   `json:"jurisdiction"`
	Action        Action  `json:"action"`
	Reasons       []string `json:"reasons"`
	EvidenceHash  string  `json:"evidence_hash"`
}

func Decide(submissionID, policyVersion, jurisdiction, evidenceHash string, e Evidence, bands map[string]Band) (Decision, error) {
	if e.SchemaVersion != "3" || e.ModelVersion == "" {
		return Decision{}, errors.New("unsupported or incomplete evidence")
	}

	action := Allow
	reasons := make([]string, 0, len(e.Scores))
	for category, score := range e.Scores {
		band, ok := bands[category]
		if !ok || math.IsNaN(score) || math.IsInf(score, 0) || score < 0 || score > 1 {
			return Decision{}, fmt.Errorf("invalid evidence for category %q", category)
		}
		if band.ReviewAt < 0 || band.BlockAt > 1 || band.ReviewAt >= band.BlockAt {
			return Decision{}, fmt.Errorf("invalid policy band for category %q", category)
		}

		switch {
		case score >= band.BlockAt:
			action = Block
			reasons = append(reasons, category+":block_band")
		case score >= band.ReviewAt && action != Block:
			action = Review
			reasons = append(reasons, category+":review_band")
		}
	}
	sort.Strings(reasons)

	return Decision{
		SubmissionID: submissionID,
		PolicyVersion: policyVersion,
		Jurisdiction: jurisdiction,
		Action: action,
		Reasons: reasons,
		EvidenceHash: evidenceHash,
	}, nil
}
```

There is a subtle exactly-once issue here. Delivery is usually at least once, so the durable invariant must be “one effective decision per submission, content hash, model version, and policy version,” enforced by an idempotency key and a uniqueness constraint. A retry may append transport attempts, but it must not create a second review task or notify the candidate twice. The decision record should include who or what decided, timestamps supplied by the storage layer, normalized reason codes, the evidence hash, and the superseded decision identifier. Store raw candidate text only where the approved retention policy permits it; a hash proves identity for reconciliation without becoming a substitute for evidence reviewers legitimately need.

## Calibrate actions against the cost of each error

Accuracy alone is a poor release criterion because it hides which errors the system makes. Build a stratified evaluation set from authorized, de-identified candidate content, including quotations, negation, reclaimed language, code, multilingual text, obfuscation, and domain terms. Split by policy category and jurisdictional workflow. Keep a frozen holdout set so repeated tuning doesn't turn the test set into training data.

For every candidate policy, calculate confusion matrices at the action level. A false block has a different consequence from an unnecessary review, while a false allow may carry a different harm by category. The operating point should therefore minimize a documented cost function under hard constraints, such as a maximum tolerated false-block rate for assessment access and a minimum recall requirement for a narrowly defined severe-harm category. Those bounds are governance decisions, not properties discovered inside a model.

Use the review queue as an instrument panel. Record overturn rate by reason code, policy version, model version, locale, and rubric task. A rising overturn rate is evidence that the decision boundary or the input distribution changed. It isn't permission to tune directly on reviewers' latest decisions: sample for disagreement, adjudicate ambiguous labels, and preserve the prior policy so backtests remain reproducible. Reviewer agreement also deserves measurement because a noisy gold label can make threshold movement look scientific when it is merely moving toward one annotator's preference.

The catch is capacity. A broad review band reduces hard false positives but can flood the queue, extend assessment completion time, and expose more sensitive content to humans. It is not suitable when the organization cannot staff trained reviewers within its declared service objective. In that case, narrow the automated policy to the highest-confidence, best-labeled categories, allow low-risk material, and defer expansion. Stick with deterministic rules for exact prohibited artifacts when context truly cannot change their meaning; use contextual classification where quotation and intent matter.

No single threshold is permanent.

## Make the queue auditable without turning it into a shadow hiring system

Reviewers need the candidate's relevant context, the matched policy text, normalized model evidence, and a constrained set of dispositions. They do not need the candidate's rubric score, demographic profile, or recruiter notes to decide a content-safety question. Removing those fields reduces bias pathways and preserves the boundary between safety review and candidate evaluation. Queue ordering should follow declared harm and aging rules rather than model confidence alone; confidence is not urgency. A review item requires an immutable creation record, lease-based assignment, an explicit disposition, and a second-review path for defined high-impact cases. Manual edits should append events because overwriting the original decision destroys the audit trail needed to explain why access was delayed or restored. Candidate notice and appeal are product states, too: `pending_review` must be distinguishable from `rejected`, and the assessment clock should follow the approved policy while review is pending. An overturned decision should trigger an idempotent restoration event, resume rubric scoring from the preserved submission, and link both decisions. This is reconciliation in its plainest form: every externally visible state must correspond to a durable event, and every durable event must have one accountable effect.

Confidence is not urgency.

Neither is silence consent.

Observability should expose aggregate rates and latency without placing raw submissions in logs. Monitor schema rejection, allow/review/block distribution, queue age, reviewer disagreement, appeal overturns, duplicate suppression, and decisions by policy version. Alert on changes against a baseline, then inspect a privacy-approved sample. A dashboard cannot prove fairness or legal compliance, but it can reveal where a targeted audit is needed.

## How should teams replay moderation decisions before changing candidate outcomes?

Start in shadow mode: run the new policy against recorded, authorized evidence and write shadow decisions without affecting candidate access. Compare old and new actions, adjudicate a stratified sample of disagreements, and obtain policy-owner approval for the resulting error trade-off. Then canary by a stable cohort key, not a random choice on every retry, while preserving an immediate route back to the prior policy version.

Promotion should require schema conformance, action-level error bounds, queue-capacity evidence, duplicate-delivery tests, reconciliation queries, retention review, and an appeal drill. Keep the old policy executable until the appeal and retention windows close. This is slower than replacing a cutoff in place, but it gives the team a defensible answer when someone asks which evidence, policy, and human judgment changed a candidate's state.

## References

- https://docs.cohere.com/docs/rerank-overview
- https://github.com/BerriAI/litellm
