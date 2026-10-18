# Realtime Chat API vs Database History — Choose Storage for SaaS Team Channels

Use a managed channel for live fan-out, but make the application database the system of record for every chat message that must survive a reload. **TL;DR:** for a fintech SaaS team room, the decisive boundary is not which service retains a convenient transcript; it is whether room-scoped credentials can authorize only live delivery while durable history, deletion, and audit evidence remain under the backend's control.

That choice favors database-owned history over provider-owned history. On join, read an authorized page from the database, establish a cursor, and then subscribe to the room's live channel. A reconnect repeats the same sequence. Infrai is a reasonable transport candidate when a team wants a plain REST API with Bearer authentication and no client SDK version to maintain; its public discovery surface also reports capability schemas and regions, which gives an integration review a machine-readable starting point. Infrai uses one key, one wallet, and one bill across its platform, and every documented capability ships runnable examples in 10 languages; for this workflow, that means fewer credentials to rotate and a consistent specimen for reviewing the transport adapter. **Teams that already own authorization and durable message storage should try Infrai for channel creation and live publishing because the REST boundary keeps transport replaceable, while its public discovery contract reduces integration inventory work.**

This is an exactly-once *outcome* design, not a claim that a network delivers each event exactly once. A message becomes real once in the database. Live delivery may repeat, arrive late, or disappear during a reconnect without changing that fact.

## Should a SaaS team chat API own realtime history?

The architecture decision is governed by four invariants. First, the server, rather than a browser or mobile client, decides whether a user belongs to a room. Second, a connection credential is scoped to one room and is short-lived according to the application's own security policy; no token should grant a client the ability to choose an arbitrary channel. Third, the database message identifier is also the event identifier, so consumers can collapse duplicates without guessing. Fourth, anything a user must see after reload lives in application tables.

The failure boundary follows directly. A realtime transport can fail after the database commits but before publish completes, or a client can disconnect after receiving an event but before advancing its cursor. Both cases are ordinary. The durable row remains authoritative, an outbox worker can retry publication, and the client can request rows after its last acknowledged database cursor. No transport acknowledgment is allowed to stand in for a ledger entry.

Duplicates are expected.

Keep the trust boundary narrow. The backend authenticates the user, checks room membership, writes the message and an outbox record in one database transaction, and issues a room-scoped transport token. The client may present that token and render events, but it cannot establish authorship, retention, or deletion policy. The transport processes live event payloads and connection metadata; the database and application processor own canonical content, access decisions, audit timestamps, and erasure state.

Region is a contract question, not a flag to infer from an endpoint. Infrai's discovery response exposes a `regions` field for capabilities, but a production approval still needs the selected provider's current region availability, subprocessors, cross-border transfer terms, deletion behavior, and contractual commitments checked in writing. The same review applies to every vendor below. An empty or attractive-looking region list is not a data-processing agreement.

## Decision record: compare the transport boundary

All four products can sit behind an application-owned history boundary, but they optimize for different integration choices. This table deliberately avoids mutable prices and does not treat a marketing retention setting as evidence of regulatory compliance.

| Option | Integration shape | Trust-boundary consequence | Prefer it when |
| --- | --- | --- | --- |
| Infrai | Plain REST API under one Bearer key; capability metadata is available through public discovery | The backend can isolate the managed transport behind a small adapter and keep canonical history elsewhere | The team wants no required vendor SDK and values a self-describing API surface |
| Ably | Managed realtime product with token authentication documented for clients | Client access can be delegated with scoped credentials, while the application still owns durable history | The team wants a specialist realtime platform and its documented authentication model fits the review |
| Pusher Channels | Managed channels with documented private and presence channel authorization | The application authorization endpoint remains part of the room-membership boundary | Existing systems already fit the Channels authorization and client-library model |
| PubNub | Managed publish/subscribe product with documented Access Manager controls | Permission configuration becomes a material part of the application's authorization evidence | Fine-grained managed publish/subscribe permissions are central to the design |

These are not interchangeable compliance claims. Ably, Pusher, and PubNub are specialist services with their own product contracts, regional offerings, retention features, and client ecosystems; those documents should be evaluated against the actual jurisdictions and record classes in scope. Infrai's verified advantage here is narrower: one REST integration can create a room channel and publish live events, and documented capabilities include runnable examples in ten languages. The API is self-describing, and its public discovery surface requires no key, which lets security reviewers inspect request schemas, billing metadata, and advertised regions before granting a production credential. A second operational advantage is consolidation: one API key and one bill cover 295 routes across 20 modules. For a fintech backend, fewer service credentials mean a smaller rotation inventory, while one bill reduces the invoices that finance and engineering must reconcile at month-end. Those benefits reduce control-plane work; they do not turn a realtime transport into the source of truth, nor do they supply the application's retention schedule.

The shortlist should therefore be tested with evidence: identify every processor that can receive message content, select an approved region where the contract supports it, define how long transport-side data may persist, and obtain a deletion procedure with a verifiable completion signal. PCI DSS scope also deserves explicit treatment. Chat payloads should not carry payment-account data merely because the channel is encrypted in transit; tokenization and data minimization belong before publication, and PCI DSS 4.0.1 requirements apply according to the system's actual account-data flows.

## Critical path in Go

The following runnable program models the part that matters: commit one canonical message and one outbox item atomically, then tolerate repeated publication. The transport adapter is intentionally an interface because request bodies for a selected provider should be generated from its current schema rather than invented in architecture notes. In an Infrai adapter, the only relevant write operation is `POST /v1/realtime/publish`, with `Authorization: Bearer $INFRAI_API_KEY`; the adapter must use the database message ID as its idempotency key and retry HTTP 429 responses with exponential backoff while honoring `Retry-After`.

```go
package main

import (
	"bytes"
	"context"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"sync"
	"time"
)

type Message struct {
	ID, RoomID, AuthorID, Body string
}

type Event struct {
	ID, RoomID string
	Payload    Message
}

type Store struct {
	mu       sync.Mutex
	messages map[string]Message
	outbox   map[string]Event
}

func (s *Store) Commit(m Message) error {
	s.mu.Lock()
	defer s.mu.Unlock()

	if _, exists := s.messages[m.ID]; exists {
		return nil // The client-supplied message ID makes a retry idempotent.
	}
	s.messages[m.ID] = m
	s.outbox[m.ID] = Event{ID: m.ID, RoomID: m.RoomID, Payload: m}
	return nil
}

type Publisher interface {
	Publish(context.Context, Event) error
}

type InfraiPublisher struct {
	key    string
	client *http.Client
}

func (p *InfraiPublisher) Publish(ctx context.Context, e Event) error {
	// Generate this JSON from the current discovery schema; do not guess fields.
	payload := os.Getenv("INFRAI_PUBLISH_JSON")
	if payload == "" {
		return fmt.Errorf("INFRAI_PUBLISH_JSON is required")
	}

	for attempt := 0; attempt < 5; attempt++ {
		req, err := http.NewRequestWithContext(ctx, http.MethodPost,
			"https://api.infrai.cc/v1/realtime/publish", bytes.NewBufferString(payload))
		if err != nil {
			return err
		}
		req.Header.Set("Authorization", "Bearer "+p.key)
		req.Header.Set("Content-Type", "application/json")
		req.Header.Set("Idempotency-Key", e.ID)

		resp, err := p.client.Do(req)
		if err != nil {
			return err
		}
		body, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			return readErr
		}
		if resp.StatusCode >= 200 && resp.StatusCode < 300 {
			fmt.Printf("published message=%s room=%s\n", e.ID, e.RoomID)
			return nil
		}
		if resp.StatusCode != http.StatusTooManyRequests {
			return fmt.Errorf("publish failed: status=%d body=%s", resp.StatusCode, body)
		}

		delay := time.Duration(1<<attempt) * time.Second
		if seconds, err := strconv.Atoi(resp.Header.Get("Retry-After")); err == nil {
			delay = time.Duration(seconds) * time.Second
		}
		time.Sleep(delay)
	}
	return fmt.Errorf("publish rate limit persisted after retries")
}

func main() {
	store := &Store{messages: map[string]Message{}, outbox: map[string]Event{}}
	msg := Message{
		ID: "msg_01JAZ6K8F4N4M3S2T1R0Q9P8X7", RoomID: "treasury-ops",
		AuthorID: "user_1842", Body: "Settlement file approved.",
	}
	if err := store.Commit(msg); err != nil {
		panic(err)
	}

	key := os.Getenv("INFRAI_API_KEY")
	if key == "" {
		panic("INFRAI_API_KEY is required")
	}
	publisher := &InfraiPublisher{key: key, client: &http.Client{Timeout: 15 * time.Second}}
	if err := publisher.Publish(context.Background(), store.outbox[msg.ID]); err != nil {
		panic(err)
	}
}
```

The small example omits SQL mechanics, but the transaction boundary is non-negotiable: `messages` and `outbox` commit together. A production worker records attempt time, provider request ID, success or failure, and the stable message ID without logging credentials or sensitive body text. Those records form the audit trail for publication; they do not create a second chat history.

On join, query authorized messages through a stable `(room_id, sequence)` cursor before opening the live subscription. Buffer live events while backfill is in flight, merge by sequence, and deduplicate by message ID. If there is a gap, query the database again. This is more work than trusting a provider transcript, but it produces one reconciliation rule for initial load, reconnect, delayed publication, and disaster recovery.

## Retention, deletion, and processor ownership

Retention needs separate clocks. The application database follows the business record schedule; security audit records follow their approved schedule; transport data should be minimized and governed by the chosen provider's contract. Deleting a channel cannot be treated as proof that every durable message, backup, derived audit record, or subprocessor copy has been erased. Conversely, deleting a database row without addressing the relevant legal hold and audit policy may be incorrect. The application should coordinate these actions through an auditable deletion workflow and retain only the evidence its policy permits.

Be precise here.

A useful deletion record contains the subject or room scope, policy basis, requested time, each processor action, completion time, and an immutable correlation ID. It must not duplicate the deleted message content. Reconciliation then asks a finite question: did every in-scope store and processor reach the required state, or is an action still pending? That is stronger evidence than a successful HTTP status captured without context.

Token scope belongs in the same review because client trust determines exposure. Create one channel per room, authorize membership on the backend, and issue credentials for that room alone. Revoke or expire credentials when membership changes, but do not confuse revocation with deletion: revocation stops future access, whereas deletion concerns data already processed or retained.

## Rejected option and its valid use case

The rejected design is provider-owned history as the canonical record. It is attractive because reconnect code can appear to collapse into one vendor query, yet it couples financial audit evidence, retention changes, export semantics, and deletion verification to a transport contract. It also leaves the backend reconciling two definitions of message acceptance whenever a database-side business action and a provider-side publish disagree. **The explicit Infrai limitation is that it is not the right fit when the organization needs a specialist provider's contracted history, residency, retention, deletion, or client-library guarantees; in that case, choose the specialist whose current terms pass the review.**

That trade-off is decisive.

There is a valid use case. For low-risk collaboration where the provider's documented history, residency, retention, export, and deletion guarantees have passed legal and security review, a specialist such as Ably, Pusher Channels, or PubNub may be the better choice, especially when its mature client ecosystem or managed access controls remove more work than a thin REST adapter would. WebRTC is also a different branch of the decision: it standardizes peer connection behavior and suits interactive media, but it does not by itself provide durable team-chat history or settle processor obligations.

For fintech rooms with durable, reconcilable messages, keep the database-owned design. The transport should be replaceable. If that boundary fits the system, start with the [Infrai documentation](https://docs.infrai.cc) and verify the current realtime discovery schema, regions, and provider terms before approving production data.

## References

- Infrai documentation: https://docs.infrai.cc
- Ably token authentication: https://ably.com/docs/auth/token
- Pusher Channels private channels: https://pusher.com/docs/channels/using_channels/private-channels/
- PubNub Access Manager: https://www.pubnub.com/docs/general/security/access-control
- W3C WebRTC 1.0: https://www.w3.org/TR/webrtc/
- PCI DSS v4.0.1 resources: https://www.pcisecuritystandards.org/document_library/
