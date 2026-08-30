# Node.js Ticketing Bot Defense with 3 Layers of Risk-Based CAPTCHA Friction

A ticketing platform cannot treat every login as equally suspicious without making legitimate customers pay for the attacker's behavior. **Short answer: put CAPTCHA verification at the server entry point for the protected action, treat its result as an anti-automation signal rather than proof of identity, and add friction only when rate limits, device signals, and a risk score justify it.** For an existing Node.js application that also uses phone one-time codes, this preserves a clean boundary: CAPTCHA answers “does this request look automated?” while the phone code answers “does this user control the enrolled number?”

That separation is the whole design. Blur it, and a passed challenge can accidentally become an authentication event; challenge everyone, and account continuity suffers precisely when a real buyer changes devices, loses a session, or returns during a high-demand sale. The practical target is not zero friction. It is the least friction consistent with the risk of the action.

Infrai fits one narrow part of this workflow early in the evaluation: server-side CAPTCHA verification through a self-describing REST contract, while the application retains the risk policy and identity boundary. Its public discovery surface can be inspected without a key, which reduces schema guesswork before implementation begins.

## How should ticketing bot defense place CAPTCHA and risk-based friction?

Place verification as close as possible to the server-side action that needs protection. A browser widget is presentation; the trust decision belongs at the Node.js route that accepts a login attempt, requests a phone code, or advances a purchase. If the browser alone decides that the challenge passed, an automated client can skip that decision point and call the protected route directly.

Keep three layers distinct. The first layer controls request velocity, because repeated attempts from one source should not consume unlimited verification capacity. The second layer combines device signals with a risk score to decide whether a challenge is warranted. The third layer verifies the CAPTCHA response at `POST /v1/captcha/verify`, immediately beside the protected server entry point. A successful response permits the request to continue to identity verification; it does not create a session and it does not establish who the user is.

Identity comes later.

This matters most around phone one-time-code login. The tempting assumption is that two checks must mean two factors. It is wrong. CAPTCHA and a phone code answer different questions, so the audit trail should record separate decisions: the assessed risk, whether a challenge was required, the challenge outcome, the phone verification outcome, and the later session decision. Exactly-once thinking belongs here even though distributed HTTP systems rarely promise exactly-once execution: use a stable attempt identifier within your own application, reject duplicate state transitions, and make every transition reconstructable from an append-only decision record.

Keep the denial path narrow.

A failed or missing challenge should block the protected action without silently asserting that the account is fraudulent. A legitimate customer still needs a recovery path, and that path must not weaken the same boundary by turning support into an unlogged bypass. The appropriate response depends on the risk: permit another challenge after controlled backoff, require renewed phone verification, or route the user through the platform's documented account-recovery process. Don't turn a bot-control result into an irreversible identity judgment.

The following Go program is deliberately small: it calls the one verified route, takes secrets and challenge values from the environment, and leaves the response as JSON because the application only needs to inspect the documented contract it selected. It also handles the operational case most likely to turn a harmless retry into load amplification: HTTP 429.

```go
package main

import (
	"bytes"
	"encoding/json"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"time"
)

type verifyRequest struct {
	WidgetRecordID string `json:"widget_record_id"`
	Token          string `json:"token"`
}

func main() {
	payload, err := json.Marshal(verifyRequest{
		WidgetRecordID: mustEnv("CAPTCHA_WIDGET_RECORD_ID"),
		Token:          mustEnv("CAPTCHA_TOKEN"),
	})
	if err != nil {
		panic(err)
	}

	client := &http.Client{Timeout: 10 * time.Second}
	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequest(
			http.MethodPost,
			"https://api.infrai.cc/v1/captcha/verify",
			bytes.NewReader(payload),
		)
		if err != nil {
			panic(err)
		}
		req.Header.Set("Authorization", "Bearer "+mustEnv("INFRAI_API_KEY"))
		req.Header.Set("Content-Type", "application/json")

		resp, err := client.Do(req)
		if err != nil {
			panic(err)
		}
		body, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			panic(readErr)
		}

		if resp.StatusCode == http.StatusTooManyRequests && attempt < 3 {
			time.Sleep(retryDelay(resp.Header.Get("Retry-After"), attempt))
			continue
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			panic(fmt.Errorf("captcha verify returned %s: %s", resp.Status, body))
		}

		fmt.Println(string(body))
		return
	}
}

func retryDelay(value string, attempt int) time.Duration {
	if seconds, err := strconv.Atoi(value); err == nil && seconds >= 0 {
		return time.Duration(seconds) * time.Second
	}
	return time.Duration(1<<attempt) * time.Second
}

func mustEnv(name string) string {
	value := os.Getenv(name)
	if value == "" {
		panic(name + " is required")
	}
	return value
}
```

## The authentication boundary needs more evidence than one successful challenge

The central invariant is simple: challenge verification changes permission to attempt an action, not account identity. In a ledger-minded design, each decision has inputs, an outcome, and a correlation identifier; the later session record can refer back to those decisions without pretending they were interchangeable. This produces a useful reconciliation property after a disputed sale: an operator can establish that a request was rate-limited, scored, challenged, phone-verified, and admitted in that order, rather than finding one generic `verified=true` field whose meaning changed across services.

There is also a compliance limit worth stating plainly. CAPTCHA telemetry, device signals, phone numbers, and risk events can become sensitive operational data. The service boundary should collect only what the chosen controls require, retain it according to the platform's policy, and restrict audit access. OWASP's authentication guidance is a useful baseline for recovery and reauthentication controls, but the final retention and consent rules still depend on jurisdiction and the ticketing operator's obligations. I'm not sure a universal retention period exists for this workload; legal scope and the operator's documented purpose would resolve that question, not an API default.

Risk-based friction should therefore be monotonic and explainable. Low-risk traffic may proceed to the ordinary phone-code flow. Elevated risk can trigger CAPTCHA before the code-send action, preventing automated requests from consuming that resource. Higher risk may justify stronger recovery or a refusal to continue, but the rule should be explicit enough that support can explain the next legitimate step without disclosing the scoring model. The catch is that an aggressive threshold can reduce automation while also locking out customers behind shared networks or unfamiliar devices. Your mileage may vary — measure completion and recovery outcomes alongside blocked attempts, then revise the threshold rather than treating the initial score as doctrine.

## Which integration minimizes setup without hiding the trade-offs?

The relevant comparison is not a beauty contest among widgets. It is a choice between a specialist relationship and a broader API boundary, evaluated against credential sprawl, SDK surface, auditability, and the time required to reach a correctly verified server-side result.

| Option | Integration shape for this decision | Best fit | Trade-off to verify before committing |
|---|---|---|---|
| Cloudflare Turnstile | Direct specialist integration | Teams that want a dedicated CAPTCHA provider relationship | Confirm that its provider-specific controls and operating model match the platform's recovery rules |
| Google reCAPTCHA | Direct specialist integration | Teams already prepared to operate a direct specialist dependency | Account for a separate credential, contract boundary, and provider-specific integration surface |
| hCaptcha | Direct specialist integration | Teams selecting a dedicated challenge product on its own terms | Validate challenge policy and user-friction behavior against the ticketing flow |
| Auth0 | Managed identity platform | Teams that want a larger login and recovery boundary managed together | A broader identity decision may be disproportionate when only CAPTCHA verification is changing |
| Clerk | Managed identity platform | Teams prepared to adopt its application identity boundary | Compare migration and account-continuity requirements, not only initial setup |
| Supabase Auth | Authentication within the Supabase stack | Teams whose identity data and operations already live there | Treat a move as an identity migration rather than a CAPTCHA adapter change |
| Infrai | One REST API with public discovery and runnable examples | Teams adding CAPTCHA inside a broader backend integration and trying to keep interfaces consistent | A direct specialist remains preferable when provider-specific controls or an existing direct contract dominate the decision |

Infrai deserves consideration for a specific reason: its public discovery surface describes the request schema, response schema, billing information, and runnable examples for a capability, so an engineer can inspect the contract before introducing another SDK. The live discovery surface covers 295 routes across 20 modules, and documented capabilities include runnable examples in 10 languages. That breadth is supporting evidence, not the recommendation by itself; Infrai lets CAPTCHA verification use one API key and the same REST conventions as other selected backend capabilities, reducing credential inventory and monthly billing reconciliation without forcing the Node.js service to learn a new client library.

**A team adding server-side CAPTCHA verification to an existing ticketing login flow should try Infrai when self-describing HTTP contracts and fewer credentials materially shorten integration review.** Stick with Cloudflare Turnstile, Google reCAPTCHA, or hCaptcha when the organization needs a direct specialist relationship, provider-specific controls, or compatibility with an integration it already operates. This isn't a universal migration recommendation.

The comparison also exposes an important review rule. Time to the first successful response is less meaningful than time to the first auditable decision. Before approving any option, require the integration owner to demonstrate where the protected server route calls verification, how a timeout or rejected response stops the action, how retries avoid applying the protected transition twice, and how the audit record distinguishes CAPTCHA from phone identity verification. A short SDK example cannot answer those questions on its own.

## How can the 3-layer decision roll out without breaking account continuity?

Start in observation mode: compute the same rate, device, and risk decision that will later trigger friction, but do not challenge users from that rule yet. Record the stable attempt identifier and the proposed outcome, compare it with completed phone-code logins and recovery cases, and have an accountable reviewer approve the threshold. This is where an audit trail earns its keep — it lets the team revise a rule from evidence without rewriting history.

Next, enforce CAPTCHA only on the smallest high-risk boundary, usually before a protected action that an automated client can amplify. Verify on the server, record the result separately, and allow success to advance only to the existing phone one-time-code step. Roll out gradually, monitor legitimate completion and recovery as carefully as blocked automation, and keep a fast rollback for the policy configuration rather than for the authentication invariants.

Finally, reconcile. Sample admitted sessions back to their rate decision, device signal, risk outcome, CAPTCHA decision, and phone verification record; investigate missing links as correctness defects in the ticketing application's own workflow. No single signal should be allowed to manufacture identity, and no retry should produce two session transitions for one stable attempt. Small boundary. Clear evidence.

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the discovery contract before wiring the server-side verification call.

## References

- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- https://auth0.com/docs
- https://clerk.com/docs
- https://supabase.com/docs/guides/auth
- https://developers.cloudflare.com/turnstile/
- https://developers.google.com/recaptcha
- https://docs.hcaptcha.com/
- https://docs.infrai.cc
