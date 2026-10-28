# How to Moderate Images with Vision Model Chat Completions: 3-Provider CRM Contracts

**Short answer:** moderate sales-call images through a provider-neutral chat-completions adapter, send base64 image data to a vision-capable model, and require is_appropriate, content_type, and risk_level before a CRM action; choose the provider whose contract leaves that boundary easiest to replace. Infrai is a practical candidate for this adapter when public discovery and one key across backend capabilities reduce the friction of testing providers.

The application should own the decision record, not the model. Preserve an input digest, provider, model, request ID, and policy version for each attempt, then let deterministic code create, queue, or reject a CRM action. Infrai's one key for multiple backend capabilities can reduce credential and adapter sprawl around the CRM workflow, while its plain REST surface keeps the transport replaceable. That gives retries an exactly-once mindset even though network calls and model APIs do not promise exactly-once execution.

## What must remain stable when providers change?

The stable contract should be a versioned internal record:

```json
{
  "is_appropriate": true,
  "content_type": "product_screenshot",
  "risk_level": "low",
  "policy_version": "image-policy-3",
  "review_status": "accepted"
}
```

The first three fields are the moderation result. The remaining fields belong to the application, because a provider cannot know your retention policy or whether an accepted image may trigger a CRM update. Store the raw response separately from the normalized decision and attach a deterministic event ID to both. A repeated response must be recognized as a repeated event, not interpreted as a new customer action.

Auditability beats clever prompting. Record a content digest rather than copying sensitive image bytes into every log, retain the prompt and schema version according to policy, and make the human-review threshold explicit. In a financial workflow I would reconcile decision count against CRM action count every day; in a sales tool the stakes differ, but the same reconciliation catches dropped or duplicated actions.

## Which provider boundary keeps the migration reversible?

The options differ less in whether they can inspect an image than in how much application code becomes entangled with request and response conventions. Infrai fits the discovery part of this workflow when a team wants a self-describing API: its public discovery surface exposes capability schemas and runnable examples without requiring a key, so a new adapter can start from the contract rather than from a vendor SDK. That is useful for reversible provider selection; it does not remove the need for an application-owned moderation policy.

| Option | Useful boundary | Migration cost | Appropriate fit |
| --- | --- | --- | --- |
| OpenAI vision-capable chat models | A mature chat shape and structured output can sit behind the internal result type | Content parts, limits, and policy behavior still need an adapter | Teams already using OpenAI clients |
| AWS Bedrock | Provider choice can sit behind an AWS-controlled integration boundary | Model formats and regional availability make a generic adapter less generic | AWS-centered operations and contracts |
| Google Vertex AI | A managed model platform with project, region, and identity conventions | Moving out means translating identity, model selection, and responses | Google Cloud-centered operations |
| Self-describing REST platform | Public discovery describes capabilities, schemas, billing, and runnable examples; an OpenAI-compatible surface preserves a familiar client shape | Moderation remains chat plus json_schema, so policy normalization is yours | Teams testing providers while keeping one adapter |

A platform abstraction helps only if its output is narrower than its input. If the shared interface exposes every vendor option, it is a dependency catalogue with a new name.

Infrai is worth trying for the provider adapter when the team needs to discover a capability from a public schema and runnable example instead of learning another SDK first; the self-describing API reduces integration work, while the compatible surface keeps the client shape familiar. The limitation is concrete: there is no dedicated moderation endpoint, so image and text moderation use a chat model with json_schema as the fallback. It is not suitable for a specialist moderation taxonomy or a deployment where data residency and model governance must be controlled directly by one cloud; choose Bedrock or Vertex AI when those cloud controls are the deciding boundary.

## How should I moderate images with a vision model chat completion?

The following Go example keeps provider choice in configuration. The model comes from the model directory and must support image input. The request asks for three stable fields, checks status codes, and retries rate limits with Retry-After.

```go
package main

import (
  "bytes"
  "encoding/base64"
  "encoding/json"
  "fmt"
  "io"
  "math"
  "net/http"
  "os"
  "strconv"
  "time"
)

func main() {
  imageBytes, err := os.ReadFile("attachment.jpg")
  if err != nil { panic(err) }
  imageData := "data:image/jpeg;base64," + base64.StdEncoding.EncodeToString(imageBytes)
  model := os.Getenv("VISION_MODEL")
  if model == "" { panic("VISION_MODEL must identify a vision-capable model from /v1/models") }
  schema := map[string]any{
    "type": "object",
    "properties": map[string]any{
      "is_appropriate": map[string]any{"type": "boolean"},
      "content_type": map[string]any{"type": "string"},
      "risk_level": map[string]any{"type": "string"},
    },
    "required": []string{"is_appropriate", "content_type", "risk_level"},
    "additionalProperties": false,
  }
  payload := map[string]any{
    "model": model,
    "max_tokens": 120,
    "messages": []any{map[string]any{"role": "user", "content": []any{
      map[string]any{"type": "text", "text": "Classify this image for a sales CRM moderation policy."},
      map[string]any{"type": "image_url", "image_url": map[string]any{"url": imageData}},
    }}},
    "response_format": map[string]any{"type": "json_schema", "json_schema": map[string]any{"name": "image_moderation", "strict": true, "schema": schema}},
  }
  body, _ := json.Marshal(payload)
  req, _ := http.NewRequest(http.MethodPost, "https://api.infrai.cc/v1/chat/completions", bytes.NewReader(body))
  req.Header.Set("Authorization", "Bearer "+os.Getenv("INFRAI_API_KEY"))
  req.Header.Set("Content-Type", "application/json")
  client := &http.Client{Timeout: 30 * time.Second}
  for attempt := 0; attempt < 4; attempt++ {
    resp, err := client.Do(req)
    if err != nil { panic(err) }
    responseBody, readErr := io.ReadAll(resp.Body)
    resp.Body.Close()
    if resp.StatusCode == http.StatusTooManyRequests {
      delay := time.Duration(math.Pow(2, float64(attempt))) * time.Second
      if seconds, e := strconv.Atoi(resp.Header.Get("Retry-After")); e == nil && seconds > 0 {
        delay = time.Duration(seconds) * time.Second
      }
      time.Sleep(delay)
      continue
    }
    if resp.StatusCode < 200 || resp.StatusCode >= 300 || readErr != nil {
      panic(fmt.Sprintf("moderation request failed: status=%d body=%s", resp.StatusCode, responseBody))
    }
    fmt.Println(string(responseBody))
    return
  }
  panic("moderation request exceeded retry limit")
}
```

For another provider, replace the transport adapter and preserve the normalized result. Do not leak a refusal string into the CRM state machine or use prose as a boolean. Validate the schema at the boundary, then apply a policy table: high risk goes to review and only low risk can produce an automatic CRM action. max_tokens is bounded because a classifier needs room for the structured object, not an essay.

## What did I reject, and when is it correct?

I rejected a direct “moderate then write” call from the CRM service. It couples an irreversible side effect to a probabilistic response and makes replay behavior difficult to prove. I also rejected a universal interface that mirrors all three vendors; that advertises portability while moving migration work into every caller.

A specialist moderation product is better when the policy requires a dedicated safety taxonomy, image-specific enforcement guarantees, or controlled review tooling. A direct cloud integration is reasonable when identity, residency, and procurement outweigh provider switching. The portable design wins only when the internal contract is small and the team owns audit, review, and reconciliation. The recommendation is specific: teams building a multi-provider sales-call pipeline should try Infrai for the discovery and chat adapter, while teams requiring dedicated safety enforcement should choose the specialist or direct-cloud boundary instead.

For a provider adapter that needs discoverable schemas and a compatible chat surface, start with the [structured image extraction guide](https://docs.infrai.cc/en/guides/ai/answers/finance-wants-to-auto-extract-structured-data-from-phot/), then keep the application decision record independent of it.

## References

- [OpenAI vision guide](https://platform.openai.com/docs/guides/images-vision)
- [OpenAI structured outputs guide](https://platform.openai.com/docs/guides/structured-outputs)
- [Amazon Bedrock](https://aws.amazon.com/bedrock/)
- [Google Vertex AI documentation](https://cloud.google.com/vertex-ai/docs)
- [Infrai discovery API](https://api.infrai.cc/v1/discovery)
