# PathFund: build plan

> Working name. Rename freely.
> Hackathon: PayPal "Build what's next with PayPal and AI" (Devpost).
> Deadline: **Nov 12, 2026, 12:00 PM PST (3:00 PM ET)**.

## How to use this file with Claude Code

Paste this as your first prompt:

```
Read PLAN.md fully. Build the app phase by phase in the order given under "Build order".
After each phase, run the app, fix errors, and tell me what works before moving on.
Do not invent PayPal API shapes: check developer.paypal.com docs for Orders v2, Payouts v1,
and JS SDK v6 before writing integration code. Keep all fee/corridor data in the seed file,
never hardcoded in components. Ask me before adding any dependency not listed in the plan.
```

---

## 1. One-line pitch

Global donors fund a cause anywhere in the world through PayPal, and an AI routing agent finds the cheapest, legal way to get that money to the cause in their local currency, then explains every fee.

## 2. Problem

- Donors can pay from almost anywhere with PayPal, but **receiving** is patchy. Zimbabwean PayPal accounts are send-only. Indian PayPal accounts auto-convert to INR at a high FX markup. In countries like Uganda, PayPal users can receive PYUSD but cannot sell it.
- The fees on a cross-border donation are mostly in the two currency conversions at the ends, not the transfer itself.
- Fundraisers do not know which rail is cheapest for their country, and they should not have to.

## 3. Solution

1. Donor pays through **PayPal Checkout** in their own currency.
2. Funds land in the platform's **US PayPal business account** (the hub).
3. The fundraiser has told us once where they want money to land (bank, PayPal, mobile wallet, spending card). They never pick a rail.
4. On withdrawal, the **AI routing agent**:
   - loads the corridor rules for the recipient country,
   - asks the route adapters for quotes on every eligible route,
   - picks the route with the highest amount delivered (tie-break on speed),
   - executes it through the route adapter,
   - writes a plain-language explanation shown to the fundraiser and donors.
5. Dashboard shows amount raised, amount delivered in local currency, and fees saved vs a standard international transfer.

## 4. Users

| User | What they do |
|---|---|
| Donor | Opens a campaign page, sees "of your $50, the cause receives about ₹4,050", pays with PayPal Checkout |
| Fundraiser | Creates a campaign, picks country, adds payout destinations, withdraws, sees agent decisions |
| Demo operator (you) | Agent console page that shows the agent's reasoning live for the video |

## 5. What is live vs simulated

Be explicit about this in the README and the video.

| Component | Status |
|---|---|
| PayPal Checkout (Orders v2, JS SDK v6) | **Live, sandbox** |
| Platform hub account | **Live, sandbox** business account |
| PayPal Payouts to fundraiser's PayPal | **Live, sandbox** |
| AI routing agent (LLM with tool use) | **Live** |
| AI campaign writer + translation | **Live** (stretch) |
| PYUSD PayPal-to-PayPal transfer | Simulated adapter |
| Xoom bank / mobile wallet / cash pickup | Simulated adapter (Xoom has no public API) |
| Stablecoin spending card | Simulated adapter (swap to Rain staging if keys are available) |
| Local crypto off-ramp | Simulated adapter |

## 6. Tech stack

| Layer | Choice | Notes |
|---|---|---|
| Framework | **Next.js 15 (App Router) + TypeScript** | Server actions and route handlers for all PayPal and LLM calls |
| UI | **Tailwind CSS + shadcn/ui** | Fast, clean. Use `lucide-react` icons |
| Database | **Supabase Postgres** | Tables in section 8. Use the service-role key only on the server |
| Auth | **Supabase Auth, email OTP** for fundraisers | Donors check out as guests. If time is tight, use a seeded demo fundraiser and skip auth |
| Storage | Supabase Storage (stretch) | Verification docs and milestone proof photos |
| Payments | **PayPal JS SDK v6** (client), **Orders v2** and **Payouts v1** REST (server) | Sandbox only |
| Agent tools | **PayPal Agent Toolkit** (`@paypal/agent-toolkit`) where it covers the call, otherwise thin REST wrappers exposed as tools | Check which tools the toolkit exposes before relying on it |
| LLM | **Anthropic Claude via `@anthropic-ai/sdk`** with tool use | Model id in env var `LLM_MODEL`. Any LLM with tool calling works |
| Validation | `zod` | Validate every LLM tool input and output |
| Hosting | **Vercel** or **Render** | Render is a hackathon sponsor (credits prize) |
| Repo | Public GitHub, **MIT license** visible in About | Required for submission |

## 7. Architecture

```
Donor browser
   │  PayPal JS SDK v6 button
   ▼
Next.js route handlers ──► PayPal Orders v2 (create, capture)  [sandbox]
   │                           │
   │                           └─► funds in Platform US business account
   ▼
Supabase (campaigns, donations, payout_methods, corridors, payouts, agent_runs)
   ▲
   │
Fundraiser "Withdraw" ──► Routing Agent (LLM + tools)
                              │
                              ├─ get_campaign_balance
                              ├─ get_corridor_rules(country)
                              ├─ list_payout_methods(fundraiser)
                              ├─ quote_routes(amount, country, methods)  ─► Route adapters
                              ├─ execute_route(route_id, quote_id)       ─► Route adapters
                              └─ record_decision(explanation)
                                                    │
               ┌───────────────┬───────────────┬────┴──────────┬──────────────┐
               ▼               ▼               ▼               ▼              ▼
        PayPalPayouts     PyusdPaypal     XoomAdapter     CardAdapter    OffRampAdapter
        (LIVE sandbox)    (mock)          (mock)          (mock / Rain)  (mock)
```

### Key design rule: the LLM never does fee math

- Fees, FX rates and eligibility come from the database and the adapters (deterministic code).
- The LLM chooses among **pre-computed, server-validated quotes** and writes the explanation.
- The server re-validates the chosen `quote_id` is eligible and unexpired before executing.
- If the LLM call fails or returns an invalid choice, fall back to the deterministic best quote and log `fallback=true`.

This is what makes the AI "meaningful" and safe, and it is worth saying in the video.

## 8. Data model (Supabase)

Create as a SQL migration in `supabase/migrations/0001_init.sql`. Enable RLS on all tables. Server code uses the service-role key.

```sql
-- Countries and what works there. Seeded, editable. Values are demo assumptions.
create table corridors (
  country_code text primary key,          -- 'IN', 'UG', 'ZW', 'PE', 'MW'
  country_name text not null,
  currency text not null,                  -- 'INR', 'UGX', ...
  fx_rate_per_usd numeric not null,        -- static demo rate
  paypal_receive text not null,            -- 'none' | 'auto_withdraw' | 'full'
  pyusd_tier text not null,                -- 'none' | 'receive_only' | 'full'
  xoom_methods text[] not null default '{}', -- {'bank','mobile_wallet','cash_pickup'}
  card_eligible boolean not null default false,
  crypto_restricted boolean not null default false,
  notes text
);

create table route_fees (
  route_id text primary key,               -- 'paypal_payouts','pyusd_paypal','xoom_bank','xoom_mobile','xoom_cash','stablecoin_card','offramp_usdc'
  display_name text not null,
  pct_fee numeric not null,                -- 0.02 = 2%
  fixed_fee_usd numeric not null,
  fx_markup numeric not null,              -- 0.03 = 3% on conversion
  eta_hours int not null,
  is_live boolean not null default false   -- true only for paypal_payouts
);

create table fundraisers (
  id uuid primary key default gen_random_uuid(),
  auth_user_id uuid,                        -- nullable for seeded demo user
  name text not null,
  email text not null,
  country_code text references corridors(country_code),
  created_at timestamptz default now()
);

create table campaigns (
  id uuid primary key default gen_random_uuid(),
  fundraiser_id uuid references fundraisers(id),
  title text not null,
  story text not null,
  story_translations jsonb default '{}',    -- {'es': '...', 'hi': '...'}
  goal_local numeric not null,
  currency text not null,
  country_code text references corridors(country_code),
  status text default 'active',
  created_at timestamptz default now()
);

create table payout_methods (
  id uuid primary key default gen_random_uuid(),
  fundraiser_id uuid references fundraisers(id),
  kind text not null,                       -- 'paypal' | 'bank' | 'mobile_wallet' | 'card' | 'cash_pickup'
  details jsonb not null,                   -- {email} | {bank_last4} | {phone, provider} ...
  is_primary boolean default false,
  created_at timestamptz default now()
);

create table donations (
  id uuid primary key default gen_random_uuid(),
  campaign_id uuid references campaigns(id),
  donor_name text,
  amount_usd numeric not null,
  paypal_order_id text unique not null,
  paypal_capture_id text,
  status text not null,                     -- 'created' | 'captured' | 'failed'
  created_at timestamptz default now()
);

create table agent_runs (
  id uuid primary key default gen_random_uuid(),
  campaign_id uuid references campaigns(id),
  amount_usd numeric not null,
  quotes jsonb not null,                    -- every quote considered
  chosen_route text,
  chosen_quote_id text,
  explanation text,
  fallback boolean default false,
  trace jsonb,                              -- tool calls, for the agent console
  created_at timestamptz default now()
);

create table payouts (
  id uuid primary key default gen_random_uuid(),
  campaign_id uuid references campaigns(id),
  agent_run_id uuid references agent_runs(id),
  route_id text references route_fees(route_id),
  amount_usd numeric not null,
  amount_local numeric not null,
  fees_usd numeric not null,
  external_ref text,                        -- PayPal payout_batch_id or mock id
  status text not null,                     -- 'pending' | 'success' | 'failed'
  is_simulated boolean not null,
  created_at timestamptz default now()
);

-- Derived view for dashboard
create view campaign_balances as
select c.id as campaign_id,
  coalesce((select sum(amount_usd) from donations d where d.campaign_id=c.id and d.status='captured'),0) as raised_usd,
  coalesce((select sum(amount_usd) from payouts p where p.campaign_id=c.id and p.status='success'),0) as paid_out_usd
from campaigns c;
```

### Seed data (`supabase/seed.sql`)

Seed 5 corridors and 7 routes. Mark everything as **illustrative**. Starting values (adjust before demo):

| Country | currency | paypal_receive | pyusd_tier | xoom_methods | card_eligible | crypto_restricted |
|---|---|---|---|---|---|---|
| India (IN) | INR | auto_withdraw | none | bank | false | true |
| Uganda (UG) | UGX | none | receive_only | mobile_wallet, bank | true | false |
| Zimbabwe (ZW) | USD | none | none | mobile_wallet, cash_pickup | false | false |
| Peru (PE) | PEN | full | full | bank | true | false |
| Malawi (MW) | MWK | full | full | bank | false | false |

| route_id | pct_fee | fixed_fee_usd | fx_markup | eta_hours | is_live |
|---|---|---|---|---|---|
| paypal_payouts | 0.044 | 0.30 | 0.035 | 24 | true |
| pyusd_paypal | 0.0 | 0.0 | 0.0 (recipient sells, see note) | 1 | false |
| xoom_bank | 0.0 | 0.0 | 0.015 | 24 | false |
| xoom_mobile | 0.0 | 0.0 | 0.015 | 1 | false |
| xoom_cash | 0.0 | 1.99 | 0.02 | 4 | false |
| stablecoin_card | 0.0 | 0.0 | 0.01 (card FX at spend) | 1 | false |
| offramp_usdc | 0.005 | 0.10 | 0.01 | 2 | false |

Eligibility rules (implemented in code, driven by corridor fields):

- `paypal_payouts`: `paypal_receive != 'none'` and fundraiser has a `paypal` method.
- `pyusd_paypal`: `pyusd_tier = 'full'` and fundraiser has a `paypal` method. Excluded when `receive_only` because the cause cannot cash out.
- `xoom_*`: matching method in `xoom_methods` and fundraiser has that method kind.
- `stablecoin_card`: `card_eligible` and not `crypto_restricted` and fundraiser has a `card` method.
- `offramp_usdc`: `pyusd_tier != 'none'` and not `crypto_restricted`. Requires a swap step, note it in the quote.

## 9. Route adapter interface

`src/lib/routes/types.ts`

```ts
export interface Quote {
  quoteId: string;
  routeId: string;
  amountUsd: number;
  feesUsd: number;            // pct + fixed
  fxCostUsd: number;          // markup on conversion
  deliveredLocal: number;     // what the cause actually gets
  currency: string;
  etaHours: number;
  isSimulated: boolean;
  steps: string[];            // human-readable, e.g. ["USD → PYUSD", "PYUSD → USDC swap", "USDC → UGX off-ramp"]
  expiresAt: string;
}

export interface RouteAdapter {
  routeId: string;
  isEligible(ctx: RouteContext): boolean;
  quote(ctx: RouteContext, amountUsd: number): Promise<Quote>;
  execute(quote: Quote, ctx: RouteContext): Promise<{ externalRef: string; status: 'success' | 'pending' | 'failed' }>;
}
```

Adapters in `src/lib/routes/`:

- `paypalPayouts.ts`: **live**. `POST /v1/payments/payouts` with `recipient_type: "EMAIL"` to the fundraiser's sandbox PayPal email. Store `payout_batch_id`.
- `pyusdPaypal.ts`, `xoom.ts` (bank, mobile, cash variants), `stablecoinCard.ts`, `offramp.ts`: **mock**. Return `mock_<route>_<nanoid>` refs, simulate a 1 to 2 second delay, always `success`.
- `registry.ts`: exports all adapters and a `quoteAll(ctx, amount)` helper.

## 10. The routing agent

`src/lib/agent/routingAgent.ts`

### Tools exposed to the LLM

| Tool | Input | Returns |
|---|---|---|
| `get_corridor_rules` | `country_code` | corridor row |
| `list_payout_methods` | `fundraiser_id` | methods (no sensitive details) |
| `quote_routes` | `campaign_id`, `amount_usd` | array of `Quote` for eligible routes, plus ineligible routes with reasons |
| `execute_route` | `quote_id` | payout result |
| `record_decision` | `chosen_quote_id`, `explanation` (≤ 60 words, plain language, include amount delivered and the runner-up) | ok |

### System prompt (starting point)

```
You are the payout routing agent for a cross-border donation platform.
Goal: deliver the most local currency to the cause, legally, using only the routes returned by quote_routes.
Process:
1. Call get_corridor_rules and list_payout_methods.
2. Call quote_routes once.
3. Pick the quote with the highest deliveredLocal. If two are within 1%, prefer the faster one.
   Never pick a route marked ineligible. Never compute fees yourself.
4. Call execute_route with the chosen quote_id.
5. Call record_decision with a short explanation for a non-technical fundraiser:
   which route, how much they receive, and how much more that is than the next option.
```

### Guardrails

- Validate every tool input with zod.
- Max 8 tool calls per run.
- Server checks `quote_id` belongs to this run and is not expired before `execute_route`.
- On any LLM error, run deterministic `pickBest(quotes)` and set `fallback=true`.
- Save the full tool-call trace to `agent_runs.trace` for the console page.

## 11. Pages

| Route | Who | Contents |
|---|---|---|
| `/` | everyone | Pitch, 3 featured campaigns (India, Uganda, Zimbabwe), "Start a fundraiser" |
| `/c/[id]` | donor | Story (language toggle if translations exist), progress bar in local currency, amount picker, **live estimate "the cause receives about ₹X"** (uses best quote), PayPal button |
| `/c/[id]/thanks` | donor | Receipt, how the money will travel (steps from best quote) |
| `/start` | fundraiser | 4-step wizard: (1) cause, story, goal, country; (2) "How do you want to receive money?" showing only eligible kinds for that country, each with "you keep about X of every 100", recommended badge; (3) connect details (PayPal sandbox email, bank last4, phone); (4) review |
| `/dashboard` | fundraiser | Raised, delivered (local), fees saved vs `paypal_payouts` baseline, donations list, payouts list with agent explanation, **Withdraw** button |
| `/agent` | demo | Pick a campaign + amount, click Run, watch the trace stream: tools called, quotes table, chosen route highlighted, explanation |

UX rules:

- Never show the words "stablecoin", "PYUSD", "off-ramp" in the fundraiser flow. Use "PayPal account", "Bank account", "Mobile wallet", "Dollar spending card", "Cash pickup". Show rails only in the "how your money travels" detail and the agent console.
- Every simulated route shows a small "Simulated in demo" badge.

## 12. API route handlers

| Method + path | Purpose |
|---|---|
| `POST /api/paypal/orders` | Create Orders v2 order for `campaign_id`, `amount_usd`. Store donation `created` |
| `POST /api/paypal/orders/[orderId]/capture` | Capture, mark donation `captured`, store `capture_id` |
| `POST /api/paypal/webhooks` | (stretch) verify signature, handle `PAYMENT.CAPTURE.COMPLETED` and `PAYMENT.PAYOUTSBATCH.SUCCESS` |
| `GET /api/quotes?campaign_id&amount_usd` | Deterministic quotes for donor estimate and wizard |
| `POST /api/agent/run` | Run routing agent for a withdrawal. Stream trace (SSE) for `/agent` |
| `POST /api/ai/story` | (stretch) Improve story + translate to 2 languages |

PayPal auth: server-side `getAccessToken()` using client credentials against `https://api-m.sandbox.paypal.com/v1/oauth2/token`, cached in memory until expiry.

## 13. Environment variables (`.env.example`)

```
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=
PAYPAL_CLIENT_ID=
PAYPAL_CLIENT_SECRET=
NEXT_PUBLIC_PAYPAL_CLIENT_ID=
PAYPAL_API_BASE=https://api-m.sandbox.paypal.com
PAYPAL_WEBHOOK_ID=
ANTHROPIC_API_KEY=
LLM_MODEL=
DEMO_MODE=true
```

## 14. Project structure

```
src/
  app/
    page.tsx
    c/[id]/page.tsx
    c/[id]/thanks/page.tsx
    start/page.tsx
    dashboard/page.tsx
    agent/page.tsx
    api/paypal/orders/route.ts
    api/paypal/orders/[orderId]/capture/route.ts
    api/paypal/webhooks/route.ts
    api/quotes/route.ts
    api/agent/run/route.ts
    api/ai/story/route.ts
  components/            # shadcn ui + PayPalButton, QuoteTable, RouteBadge, AgentTrace
  lib/
    supabase/server.ts
    supabase/client.ts
    paypal/auth.ts
    paypal/orders.ts
    paypal/payouts.ts
    routes/types.ts
    routes/registry.ts
    routes/eligibility.ts
    routes/adapters/*.ts
    agent/routingAgent.ts
    agent/tools.ts
    agent/prompt.ts
supabase/
  migrations/0001_init.sql
  seed.sql
README.md
LICENSE (MIT)
.env.example
```

## 15. Build order (time-boxed, ~16 hours of work)

| # | Phase | Box | Done when |
|---|---|---|---|
| 0 | Setup: Next.js, Tailwind, shadcn, Supabase project, migration, seed, env, PayPal sandbox app (business + personal accounts, Payouts enabled) | 1h | App runs, tables seeded |
| 1 | Route engine: types, eligibility, all adapters (Payouts live, rest mock), `quoteAll`, unit-test with a script for IN, UG, ZW | 2h | Script prints sensible quotes per country |
| 2 | Donor flow: campaign page, estimate, PayPal button, create + capture order, donation saved | 2.5h | Real sandbox payment shows in dashboard |
| 3 | Routing agent: tools, prompt, guardrails, fallback, `agent_runs` logging, `/api/agent/run` | 2.5h | Agent picks Xoom bank for IN, mobile wallet for UG, executes, logs |
| 4 | Fundraiser: `/start` wizard (country-filtered options), `/dashboard` with Withdraw and explanations | 2.5h | End-to-end: create campaign → donate → withdraw → explanation |
| 5 | `/agent` console with streamed trace and quote table | 1.5h | Looks good on video |
| 6 | Polish, seed 3 demo campaigns, README, LICENSE, deploy | 1.5h | Public URL works |
| 7 | Video + Devpost submission | 1.5h | Submitted before 2:00 PM ET buffer |

### Cut list if behind (cut from the top)

1. Webhooks (rely on capture response)
2. AI story writer + translations
3. Supabase Auth (use seeded demo fundraiser)
4. SSE streaming on `/agent` (render trace after run completes)
5. Rain staging card (keep mock)

Never cut: live PayPal Checkout, live Payouts, the agent with tool use, the explanation text.

## 16. Demo video script (under 3 minutes)

1. **0:00 to 0:20** Problem: same $100 donation, a cause in Zimbabwe cannot receive PayPal, India loses ~8% to fees and FX, Uganda can receive PYUSD but cannot cash it out.
2. **0:20 to 0:50** Fundraiser in Uganda creates a campaign. Wizard shows only Mobile wallet, Bank, Dollar card. No crypto jargon.
3. **0:50 to 1:20** Donor in the US opens the page, sees "the cause receives about UGX X", pays with PayPal sandbox. Donation appears.
4. **1:20 to 2:10** `/agent` console: run withdrawal for India and Uganda side by side. Show quotes, the chosen route, and the explanation. Point out the LLM never does fee math.
5. **2:10 to 2:40** Dashboard: delivered in local currency, fees saved, live PayPal Payout batch id.
6. **2:40 to 2:55** Live vs simulated, and production path: licensed partners for Xoom, PYUSD and card issuing. Funds are never used by the platform.

## 17. Submission checklist

- [ ] Public GitHub repo, MIT `LICENSE`, license visible in repo About
- [ ] README: pitch, architecture diagram, live vs simulated table, setup and run steps, seed instructions, sandbox account setup
- [ ] "AI tools used" section: Claude via Anthropic SDK (routing agent with tool use, explanations, translation), PayPal Agent Toolkit if used, Claude Code for development
- [ ] Hosted demo URL or clear run instructions
- [ ] Video under 3 minutes, public on YouTube, no copyrighted music, no unlicensed third-party logos
- [ ] Devpost text description

## 18. Compliance notes (put in README)

- Sandbox only. No real funds.
- Corridor rules and fees are illustrative and must be verified per country before production.
- In production, funds would be held and moved by licensed partners (PayPal, Xoom, a licensed card issuer). The platform never lends, invests or otherwise uses donor or fundraiser funds.
- Platform revenue model: disclosed platform fee and card interchange share. Any yield on held PYUSD is passed to the cause.
