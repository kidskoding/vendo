# Vendo — Design

**Date:** 2026-08-09
**Status:** Approved, ready for implementation planning

## Purpose

A Rails application that exposes a product catalog and checkout flow to AI shopping
agents via the Agentic Commerce Protocol (ACP), plus a Claude-powered buyer-agent
harness that exercises it end to end.

The immediate goal is a learning and portfolio artifact. The project is expected to
evolve along a known path:

1. Learning / portfolio piece (this design)
2. A real store that agents can buy from
3. An open-source engine other merchants mount into their Rails stores
4. A commercial product

Step 1 is what gets built. The design's job is to make steps 2 and 3 cheap rather
than to implement them now.

## Background: what ACP is and why the merchant renders nothing

ACP is the Agentic Commerce Protocol (OpenAI + Stripe). Under it, the buyer never
visits the merchant's site. The human talks to an agent (ChatGPT, or in this project
a local harness), the agent reads the merchant's machine-readable product feed,
and the agent calls the merchant's checkout API on the human's behalf. The agent's
interface replaces the storefront.

```
human ──chats──> buyer agent
                     │
                     │ reads product feed        ┌──────────────┐
                     ├──────────────────────────>│              │
                     │ POST /checkout_sessions   │  RAILS APP   │
                     ├──────────────────────────>│  (merchant)  │
                     │ ...complete + token       │              │
                     └──────────────────────────>│              │
                                                 └──────┬───────┘
                                                        ▲
                                   merchant operator ───┘
```

Consequences for this design:

- No public storefront. The merchant app serves a feed and an API.
- The only human-facing UI is the **merchant admin** — for the operator, not the buyer.
- Payment is *delegated*: the agent's payment provider vaults the card and hands the
  merchant a shared payment token, which the merchant charges.

## Scope

**In scope**

- ACP product feed endpoint
- ACP Checkout Sessions API (create, retrieve, update, complete, cancel)
- Delegated payment handling behind a processor seam
- Bearer-token authentication for agent clients
- Idempotency on all POST endpoints
- Merchant admin: product management, live checkout-session and order views
- Buyer-agent harness (CLI, Claude tool-use loop)
- Order-update webhooks, stubbed (fire and log)

**Out of scope**

- Public human-facing storefront
- Multi-tenancy
- Request signature verification
- Webhook retry/delivery guarantees
- Real tax and carrier shipping rates
- MCP or AP2 protocol surfaces

## Architecture

```
buyer agent (harness/, CLI)          merchant app (Rails)
  │                                    ┌────────────────────────────┐
  │  Claude + 4 tools                  │ Vendo:: controllers        │
  │                                    │   ├ FeedController         │
  ├── GET  /feed.json ────────────────▶│   └ CheckoutSessions       │
  ├── POST /checkout_sessions ────────▶│         │                  │
  ├── POST /checkout_sessions/:id ────▶│         ▼                  │
  └── POST /checkout_sessions/:id/     │  Vendo::Catalog  ◀── SEAM  │
           complete ─────────────────▶ │         │                  │
                                       │         ▼                  │
                                       │   Product / CheckoutSession│
                                       │   / LineItem / Order       │
                                       └────────────┬───────────────┘
                                                    ▼
                                       Admin::  (operator watches here)
```

### The seam rule

`Vendo::Catalog` is the only file permitted to reference `Product`, `LineItem`, or
`Order`. ACP controllers call `Catalog.feed_items`, `Catalog.build_session(...)`,
`Catalog.in_stock?`, `Catalog.decrement!`, and never touch ActiveRecord directly.

This is one module of roughly sixty methods-worth of plain Ruby — no interface, no
dependency injection, no registry. It exists for one reason: at step 3, the ACP
layer moves into a mountable engine and the host application supplies its own
`Catalog`. Everything above the seam moves unchanged.

The seam also resolves multi-tenancy. There is deliberately **no `Merchant` model
and no `merchant_id` column**. Because every query routes through `Catalog`, adding
tenancy later is a change to one file rather than a migration and query-rewrite
across five. A `ponytail:` comment in `Catalog` records this.

### Alternatives considered

**Flat Rails, no seam.** ACP controllers touch models directly. Fewest files, fastest
to something running. Rejected because engine extraction at step 3 becomes a rewrite —
an engine cannot know the host application's model names, so every controller needs
rework.

**In-repo mountable engine from day one.** Real boundary enforced by Rails
`isolate_namespace`. Rejected for now: it pays the step-3 cost during step 1 (dummy
app for tests, second Gemfile, route mounting, constant-lookup friction in every
file) and engine ceremony is the most likely reason for the project to stall. The
chosen approach converts to this cheaply when distribution actually happens.

## Data model

All monetary values are **integer minor units** (cents). No `Money` gem, no
`BigDecimal`.

| Model | Fields |
|---|---|
| `Product` | `item_id`, `title`, `description`, `url`, `image_url`, `price_cents`, `currency`, `availability`, `inventory_quantity`, `group_id`, `brand`, `condition` |
| `CheckoutSession` | `public_id`, `terminal_status` (nil/`completed`/`canceled`), `buyer` (json), `fulfillment_address` (json), `fulfillment_option_id`, `currency` |
| `LineItem` | `checkout_session_id`, `product_id`, `quantity`, `base_amount`, `discount`, `subtotal`, `tax`, `total` |
| `Order` | `checkout_session_id`, `status`, `payment_reference`, `permalink_url` |
| `IdempotencyKey` | `key` (unique index), `request_fingerprint`, `response_body`, `status_code` |
| `ApiClient` | `name`, `token_digest`, `webhook_url` |

Product variants are represented by `group_id` on a flat `Product` table, as the
feed spec does. There is no separate `Variant` model.

`price_cents` + `currency` are the storage form. The feed emits the spec's
`"79.99 USD"` string; the checkout API emits integer minor units. One formatter
each way, no `Money` gem.

Feed fields with no column — `seller_name`, `seller_url`, `target_countries`,
`store_country`, `is_eligible_search`, `is_eligible_checkout`, `seller_tos`,
`seller_privacy_policy` — are required by the spec but identical for every row in
a single-merchant store. They come from `Rails.configuration.vendo`, not the
database.

`ponytail:` seller fields as config — becomes a `Merchant` record at step 3, when
the engine has multiple hosts. Same argument as the `Catalog` seam.

### Status is computed, not tracked

`CheckoutSession#status` is derived, not driven by stored transitions. It returns
`ready_for_payment` when a fulfillment address is present, a fulfillment option is
selected, and every line item is in stock; `not_ready_for_payment` otherwise.
`completed` and `canceled` are the two states that *are* stored, because they are
terminal facts rather than derivations. No state-machine gem.

### Inventory is checked, not held

Creating or updating a session **does not reserve stock**. `Catalog.in_stock?`
reads `inventory_quantity` for the status computation; `Catalog.decrement!` runs
inside the completing transaction and re-checks, so a session that went stale
fails at completion rather than overselling.

`ponytail:` no reservation — two agents can both hold a `ready_for_payment`
session for the last unit and one loses at completion. Real holds need a
`Reservation` row with a TTL and a sweeper. Add when a real store has contention.

## Request handling

`Vendo::BaseController` performs four steps in order:

1. **Bearer authentication** against `ApiClient#token_digest`.
2. **Idempotency.** On a POST carrying an `Idempotency-Key`, look up the key. On a
   hit with a matching request fingerprint, replay the stored response body and
   status verbatim. On a hit with a differing fingerprint, return the ACP
   `request_not_idempotent` error. On a miss, process and store the result.
3. **Header echo.** Reflect `Idempotency-Key` and `Request-Id` back on the
   response, and set `API-Version: 2025-09-12`. Three lines in an `after_action`.
4. **Error rendering** in ACP's error shape: `{type, code, message, param}`, where
   `type` is `invalid_request` and `code` is drawn from the spec's enum
   (`request_not_idempotent` is the only one this app emits at the top level —
   per-field problems go in `messages`, not here).

Inbound `Signature`, `Timestamp`, and `Accept-Language` are accepted and ignored.

Serialization is a plain Ruby object returning a Hash. No jbuilder, no serializer gem.

Note that ACP nests the ordered item inside the line item: a `LineItem`
serializes as `{id, item: {id, quantity}, base_amount, discount, subtotal, tax,
total}`. The database keeps `product_id` and `quantity` flat on the row; the
serializer does the nesting.

### Totals

ACP's `totals` is an **array** of `{type, display_text, amount}`, not a hash. The
`type` enum is `items_base_amount`, `items_discount`, `subtotal`, `discount`,
`fulfillment`, `tax`, `fee`, `total`. `Vendo::Totals` emits the four the store
actually uses — `items_base_amount`, `fulfillment`, `tax`, `total` — and omits the
rest rather than emitting zeros.

Tax is a flat configured rate; shipping is the selected fulfillment option's
`total`.

`ponytail:` flat tax rate — real tax needs a tax service (Avalara, Stripe Tax).
Swap when there is a real store.

### Fulfillment options

`fulfillment_options` is required in **every** checkout-session response, so it
cannot be skipped even in a store with one shipping method. `Vendo::Fulfillment`
returns a single hardcoded `type: "shipping"` option built from config — id,
title, subtitle, carrier, a delivery window computed as *now + N days*, and the
flat shipping amount.

`ponytail:` one hardcoded shipping option with a computed window — real carrier
rates and delivery estimates need a rates API. Same swap point as tax.

### Messages and links

`messages` and `links` are required in every response and are easy to miss.
`messages` is `[]` on a clean session and carries `{type: "error", code:
"out_of_stock", param: "$.line_items[0]", content_type: "plain", content: ...}`
when a line item cannot be fulfilled — this is how the agent learns *why* a
session is `not_ready_for_payment`, so it is load-bearing for the demo, not
decoration. `links` is a static list of `terms_of_use` and `privacy_policy` URLs
from config.

`payment_provider` (`{provider: "stripe", supported_payment_methods: ["card"]}`)
is required on the create and complete responses only.

### Payments

```ruby
Vendo::PaymentProcessor.charge(token:, amount_cents:)
```

`FakeProcessor` (default) returns a synthetic payment reference. `StripeProcessor`
sits behind the identical call and goes unused until ACP-enabled Stripe credentials
exist. This is not a speculative abstraction: real shared payment tokens are
unobtainable in development, so a fake is required for the system to run at all, and
the real implementation is a known, named second case.

### Webhooks

On order state change, POST to `ApiClient#webhook_url` and log the result. The
body is the spec's event shape:

```ruby
{ type: "order_created",              # or "order_updated"
  data: { type: "order",
          checkout_session_id: ...,
          permalink_url: ...,
          status: "created",          # created|manual_review|confirmed|canceled|shipped|fulfilled
          refunds: [] } }
```

No retry queue, no signing, no delivery guarantees.

`ponytail:` fire-and-log webhooks — add a background job with retries and
`{Merchant}-Signature` HMAC generation when a real merchant depends on delivery.

## Buyer-agent harness

Plain Ruby under `harness/`, outside `app/`, so it is not Rails-autoloaded. Invoked
as `bin/shop "buy me a blue mug"`.

The agent is given four tools and no scripted sequence — it reads ACP-shaped
responses and works out the flow itself. That is the demonstration.

```ruby
class CreateCheckoutSession < Anthropic::BaseTool
  doc "Create an ACP checkout session with the given item IDs and quantities."
  input_schema CreateCheckoutInput
  def call(input) = MerchantHttp.post("/checkout_sessions", ...)
end

client.beta.messages.tool_runner(
  model: :"claude-opus-5",
  max_tokens: 8192,
  output_config: { effort: "medium" },
  tools: [
    FetchFeed.new,
    CreateCheckoutSession.new,
    UpdateCheckoutSession.new,
    CompleteCheckoutSession.new
  ],
  messages: [{ role: "user", content: request }]
).each_message { |message| render(message) }
```

`effort: "medium"` because shopping is not a hard reasoning task and the harness runs
frequently during development.

The harness has no chat UI. It runs in one terminal pane while the merchant admin is
open in a browser, so the operator watches the agent buy in real time.

## Merchant admin

Rails scaffolding for products, plus a checkout-sessions and orders view. Its purpose
is to make the demo legible to a human — the agent's activity appears as rows
arriving while the harness runs.

## Testing

Three Minitest integration tests, using Rails defaults with no additional framework:

1. **Full happy path** — feed → create session → update with address → complete →
   `Order` exists with the expected totals.
2. **Idempotency replay** — the same key posted twice produces one order and two
   byte-identical responses.
3. **Totals math** — subtotal plus tax plus shipping.

The harness is deliberately excluded from CI: it costs LLM calls on every run. It is
the demo; the tests are the correctness check.

## Spec reconciliation (resolved 2026-08-09)

This design was originally written from memory and flagged its own field names as
unverified. They have since been reconciled against the published ACP specs
(`developers.openai.com/commerce/specs/checkout` and `/specs/feed`, API version
`2025-09-12`). The field names, enum values, and header names above are now taken
from those documents rather than recalled.

The expectation that corrections would be "field-name level, not structural" held
for the feed and for the error shape, but **not** for the checkout session, which
needed four structural additions:

- `fulfillment_options` is required in every response — there was no such concept
- `totals` is a typed array, not a hash
- `messages` and `links` are required in every response
- `payment_provider` is required on create and complete

These are reflected above. The seam, the model list, the computed-status decision,
and the processor boundary are unchanged.

## Deliberate simplifications

Each is marked with a `ponytail:` comment at its site in the code:

| Simplification | Ceiling | Upgrade path |
|---|---|---|
| Flat tax rate | Wrong for real jurisdictions | Tax service |
| Flat shipping amount | Wrong for real carriers | Carrier rate API |
| Fake payment processor default | Cannot take real money | `StripeProcessor` behind the same call |
| Fire-and-log webhooks | Silent delivery failures | Background job with retries + signing |
| No request signature verification | Bearer token is the only auth | ACP `Signature` header verification |
| No multi-tenancy | Single merchant only | Scope inside `Vendo::Catalog` |

## Open decisions deferred

None blocking implementation. The engine extraction (step 3) is a deliberate
follow-on, not a gap in this design.
