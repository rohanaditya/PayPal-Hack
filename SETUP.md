# PathFund: base setup and team split

Goal: in about 90 minutes, have a repo with the database, folder structure, shared types and API stubs in place, so the team can split up and build in parallel without stepping on each other.

Deadline reminder: submissions close **Nov 12, 12:00 PM PST (3:00 PM ET)**.

---

## Part A. Manual steps (one person, ~25 min)

Do these yourself. Claude Code can't create accounts for you.

### A1. GitHub
1. Create a **public** repo, e.g. `pathfund`. Initialise it with a README, and choose **MIT License** in the creation form so it shows in the repo's About section.
2. Add teammates as collaborators.
3. Clone it locally and copy `PLAN.md` and this `SETUP.md` into the root.

### A2. Supabase
1. Create a new project at supabase.com. Note the region and DB password.
2. From **Project Settings → API**, copy:
   - Project URL → `NEXT_PUBLIC_SUPABASE_URL`
   - `anon` public key → `NEXT_PUBLIC_SUPABASE_ANON_KEY`
   - `service_role` key → `SUPABASE_SERVICE_ROLE_KEY` (server only, never commit)
3. You'll run the migration and seed SQL in the **SQL Editor** after prompt 2 below. That's faster than linking the CLI tonight.

### A3. PayPal sandbox
1. Log in at developer.paypal.com and switch to **Sandbox**.
2. **Apps & Credentials → Create App** (type: Merchant). Copy the client ID and secret.
3. In the app's features, confirm **Payouts** is enabled.
4. **Testing Tools → Sandbox Accounts**. You should already have a default Business account (this is your **platform hub**) and a Personal account (use it as the **donor**). Create two more Personal accounts to act as fundraiser PayPal accounts:
   - `fundraiser-in@...` with country **India**
   - `fundraiser-ug@...` with country **Uganda** (if Uganda isn't offered, use any country and note it)
5. Save all sandbox emails and passwords in a shared team note, not in the repo.

### A4. LLM key
Create an Anthropic API key (or whichever LLM your team prefers). You'll set the model ID in `LLM_MODEL`.

### A5. Local `.env.local`
After prompt 1 creates `.env.example`, copy it to `.env.local` and fill in the values. Share `.env.local` with teammates privately, not through git.

---

## Part B. Claude Code prompts (run in order, one person)

Run these one at a time from the repo root. Wait for each to finish, check the "Done when" line, then commit.

### Prompt 1. Scaffold the project

```
Read PLAN.md fully, especially sections 6, 13 and 14.

Scaffold the project in the current repo root (it already has README.md, LICENSE, PLAN.md, SETUP.md; keep them):
- Latest stable Next.js with App Router, TypeScript, Tailwind, ESLint, src/ directory, import alias "@/*".
- Initialise shadcn/ui and add: button, card, input, label, select, badge, table, tabs, progress, dialog, toast (or sonner), separator, skeleton.
- Install: @supabase/supabase-js, @supabase/ssr, @anthropic-ai/sdk, zod, nanoid, lucide-react, @paypal/paypal-js. Add tsx as a dev dependency for scripts.
- Create every folder and file from PLAN.md section 14 as a stub. Each stub page renders its route name in a shadcn Card. Each stub lib file exports placeholder functions with TODO comments and correct names.
- Create .env.example exactly as in PLAN.md section 13. Add .env.local to .gitignore.
- Add npm scripts: "dev", "build", "lint", "typecheck" (tsc --noEmit), "script" (tsx).
- Replace README.md with a skeleton: pitch, architecture placeholder, "Live vs simulated" table from PLAN.md section 5, setup steps, AI tools used, compliance notes from section 18.

Do not implement business logic yet. Run typecheck and build and fix any errors.
```

**Done when:** `npm run dev` shows every page from section 11 as a stub, and `npm run build` passes. Commit: `chore: scaffold project`.

### Prompt 2. Database

```
Using PLAN.md section 8:
1. Write supabase/migrations/0001_init.sql with all tables and the campaign_balances view. Enable RLS on every table and add no policies (all access goes through the server with the service-role key).
2. Write supabase/seed.sql:
   - corridors: IN, UG, ZW, PE, MW using the table in section 8. Static fx_rate_per_usd: IN 84, UG 3700, ZW 1, PE 3.75, MW 1730. Put "Illustrative demo values" in notes.
   - route_fees: all 7 routes from section 8. Only paypal_payouts has is_live = true.
   - One demo fundraiser per country for IN and UG, with emails matching our sandbox accounts (use placeholders FUNDRAISER_IN_EMAIL and FUNDRAISER_UG_EMAIL that I'll replace).
   - Three demo campaigns: one in India, one in Uganda, one in Zimbabwe, with realistic short stories and goals in local currency.
   - Payout methods: India fundraiser has paypal + bank. Uganda fundraiser has mobile_wallet + card + bank.
3. Write src/lib/supabase/server.ts (service-role client, server-only, throws if imported client-side) and src/lib/supabase/client.ts (anon client).
4. Write src/lib/db/types.ts with TypeScript row types for every table, matching the SQL exactly.
5. Write scripts/check-db.ts that connects with the service role and prints row counts for every table.

Tell me the exact order to paste the files into the Supabase SQL Editor.
```

**Then, manually:** paste `0001_init.sql`, then `seed.sql`, into the Supabase SQL Editor and run them. Replace the placeholder emails with your sandbox emails.

**Done when:** `npm run script scripts/check-db.ts` prints 5 corridors, 7 routes, 3 campaigns. Commit: `feat: database schema and seed`.

### Prompt 3. Shared contracts (the most important step for splitting work)

```
Create the shared contracts so four people can build in parallel against fixed interfaces.

1. src/lib/routes/types.ts: Quote, RouteAdapter, RouteContext exactly as PLAN.md section 9.
   RouteContext = { campaign, fundraiser, corridor, payoutMethods } using the row types from src/lib/db/types.ts.
2. src/lib/routes/eligibility.ts: function signatures only, isRouteEligible(routeId, ctx) and ineligibleReason(routeId, ctx). Implement the rules from PLAN.md section 8 "Eligibility rules" since they're small and everyone depends on them.
3. src/lib/routes/adapters/: one file per route_id with a class implementing RouteAdapter. Mock adapters can be fully implemented now (deterministic fee math from route_fees, fake refs mock_<route>_<nanoid>, 1 second delay). Leave paypalPayouts.execute as a TODO that throws "not implemented".
4. src/lib/routes/registry.ts: export all adapters and quoteAll(ctx, amountUsd) returning { eligible: Quote[], ineligible: { routeId, reason }[] }.
5. src/lib/schemas.ts: zod schemas for every API request and response in PLAN.md section 12.
6. Every API route handler from section 12: parse input with the zod schema, return 501 with { error: "not implemented" } typed to the response schema. /api/quotes should actually work using quoteAll.
7. src/lib/agent/tools.ts: tool definitions (names, descriptions, zod input schemas) from PLAN.md section 10, with handler functions as TODOs.
8. docs/CONTRACTS.md: a one-page summary of every interface, schema and endpoint, and which file owns it.

Run typecheck and build. Fix errors.
```

**Done when:** `GET /api/quotes?campaign_id=<india id>&amount_usd=100` returns quotes for `xoom_bank` and `paypal_payouts`, with the other routes listed as ineligible with reasons. Commit: `feat: shared contracts and route engine`.

### Prompt 4. PayPal sandbox smoke test

```
Implement src/lib/paypal/auth.ts: getAccessToken() using client credentials against PAYPAL_API_BASE, cached in memory until expiry.
Write scripts/paypal-smoke.ts that:
1. gets a token,
2. creates a $1.00 USD Orders v2 order with intent CAPTURE and prints its id and approve link,
3. sends a $1.00 sandbox Payout to FUNDRAISER_IN_EMAIL from .env.local and prints payout_batch_id and batch status.
Check developer.paypal.com docs for the exact Orders v2 and Payouts v1 request shapes before writing them. Do not wire this into the app yet.
```

**Done when:** the script prints an order ID and a payout batch ID. If the payout fails, check that Payouts is enabled on the sandbox app before debugging code. Commit: `chore: paypal sandbox smoke test`.

### Prompt 5. Test fixtures for the 10 transactions

```
Create tests/fixtures/transactions.ts with these 10 cases. Each case: id, donorCountry, causeCountry, donorPays (amount + currency), hubAmountUsd, payoutMethods, expected { decision: 'execute' | 'hold' | 'failover' | 'fallback', routeId?, deliveredLocal? }.

Donor-side conversion: non-USD donors lose 3% at checkout. Rates GBP 1.27, EUR 1.08, CAD 0.73, SGD 0.75, INR 84 per USD.

1. US→IN, $100, hub 100.00, [bank, paypal] → execute xoom_bank, ₹8,274
2. UK→IN, £80, hub 98.55, [paypal] → execute paypal_payouts, ₹7,613, with nudge to add bank
3. US→IN, $1000, hub 1000.00, [bank, paypal] → execute xoom_bank, ₹82,740
4. CA→IN, C$70, hub 49.57, [card] → hold, no eligible route
5. DE→IN, €90, hub 94.28, [bank, paypal], xoom adapter forced to fail → failover to paypal_payouts, ₹7,282
6. US→UG, $100, hub 100.00, [mobile_wallet, card, bank] → execute stablecoin_card, UGX 366,300
7. SG→UG, S$130, hub 94.58, [paypal] → hold, no eligible route
8. IN→UG, ₹1700, hub 19.63, [mobile_wallet] → execute xoom_mobile, UGX 71,545
9. UK→UG, £400, hub 492.76, [bank, mobile_wallet] → execute xoom_mobile, UGX 1,795,864 (tie broken on speed)
10. US→UG, $100, hub 100.00, [mobile_wallet, card], LLM returns invalid route "pyusd_paypal" → fallback stablecoin_card, UGX 366,300

Write scripts/run-fixtures.ts that, for each case, builds a RouteContext from the fixture, calls quoteAll, and checks that the deterministic best quote matches the expected route and deliveredLocal (allow ±1 rounding). Cases 4, 5, 7 and 10 need agent behaviour, so for now just check that quoteAll returns the right eligible/ineligible sets. Print a pass/fail table.
Add support for forcing an adapter to fail via the context (needed for case 5).
```

**Done when:** every case passes its current-stage checks. If a number is off, fix the seed or the fixture, not the test. Commit: `test: transaction fixtures`.

### After setup

Push to `main`. From here, everyone works on their own branch (`feat/<area>`) and opens a PR to `main`. Keep PRs small, and merge often.

---

## Part C. Team split

Everything below builds on the contracts from prompt 3. Only touch files your area owns. If you need a contract change, post it in the team chat first, because others depend on it.

| Area | Owner | Owns these files | Depends on | Done when |
|---|---|---|---|---|
| **1. Donor checkout** | Person A | `app/c/[id]/*`, `api/paypal/orders/*`, `lib/paypal/orders.ts`, `components/PayPalButton.tsx` | Prompts 3, 4 | A sandbox donor pays, the donation is saved as `captured`, and the page shows "cause receives about X" from `/api/quotes` |
| **2. Routing agent** | Person B (you) | `lib/agent/*`, `api/agent/run`, `lib/paypal/payouts.ts`, `paypalPayouts.execute` | Prompts 3, 4, 5 | All 10 fixtures pass through the full agent, including hold, failover and fallback. Agent runs are logged to `agent_runs` |
| **3. Fundraiser app** | Person C | `app/start/*`, `app/dashboard/*`, payout method forms | Prompt 3 | The wizard shows only eligible methods for the chosen country, and the dashboard shows raised, delivered, fees saved, and a Withdraw button that calls `/api/agent/run` |
| **4. Agent console, landing, demo** | Person D | `app/agent/*`, `app/page.tsx`, `components/AgentTrace.tsx`, `QuoteTable.tsx`, README, video | Prompt 3 (use the `agent_runs` shape, with mock data until area 2 lands) | The console shows the tool trace, quote table and explanation. README complete. Video script ready |

With fewer people, merge areas 3 and 4. With two people, one takes areas 1 and 3 (everything users click), the other takes areas 2 and 4 (agent and demo).

### Suggested Claude Code prompt for each teammate

```
Read PLAN.md, SETUP.md and docs/CONTRACTS.md. I own area <N> in SETUP.md Part C.
Only modify files listed for my area. If I need a contract change, stop and tell me instead of making it.
Build my area until its "Done when" condition is met, running typecheck and build as you go.
```

### Integration checkpoints (ET)

| Time | Checkpoint |
|---|---|
| Tonight, before sleep | Setup prompts 1 to 5 merged to `main` |
| 9:00 AM | Each area works on its own branch. Quick sync |
| 11:00 AM | Everything merged. End-to-end run: donate → withdraw → agent → payout |
| 12:00 PM | Deploy (Vercel or Render). Freeze features, bug fixes only |
| 12:30 PM | Record video |
| 2:00 PM | Submit on Devpost. One-hour buffer before the 3:00 PM deadline |
