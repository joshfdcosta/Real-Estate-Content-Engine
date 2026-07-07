# Real Estate Content Engine — Implementation Plan

## 1. Architecture Summary

**Single most important decision (solo dev):**
Do not split frontend and backend into separate services. For a solo developer, running a separate FastAPI or Node backend doubles deploys, auth plumbing, and cognitive load for no MVP benefit. Build a single Next.js app with route handlers as your API. You can extract a separate service later if a specific workload demands it (heavy scraping in Phase 2 is the only likely candidate).

**Recommended stack (primary path):**

- **Framework:** Next.js (App Router) — full-stack. React UI plus route handlers under `app/api/*` as the backend. Use Server Actions for simple mutations, and route handlers for anything you will test with `curl` or call from a webhook.
- **Database:** PostgreSQL on Neon or Supabase.
  - Pick Supabase if you want auth + storage + DB bundled.
  - Pick Neon + Auth.js if you prefer keeping auth framework-native.
  - This plan assumes **Neon + Prisma + Auth.js**, with notes for Supabase where relevant.
- **ORM:** Prisma (fastest for a solo dev; migrations + typed client). Drizzle is a lighter alternative if you prefer SQL-first.
- **Auth:** Auth.js (NextAuth) with Google OAuth + email magic link.
- **AI:** Claude API behind a thin `lib/ai` wrapper.
  - Prompt templates per asset type.
  - Structured JSON output.
  - Zod validation and retries.
  - Confirm the exact model string and current pricing/limits against docs.claude.com when wiring it up; start with a Sonnet-class model for the cost/quality balance a content generator needs.
- **Billing:** Stripe Checkout + Customer Portal + webhooks. Plans and quotas enforced in your own DB.
- **Hosting:** Vercel (app) + Neon/Supabase (DB). One deploy target.
- **Exports (MVP):** Copyable text and Markdown download first. Add PDF later (serverless PDF rendering is a rabbit hole).

**Where external APIs fit:**

- Nowhere in Phase 1.
- All property-data, market-stats, and social integrations sit behind adapter interfaces introduced in Phase 2, so the MVP never depends on a third party you do not control.

**LLM latency trap:**

A full content pack requires many LLM calls (captions × 3 platforms, reel scripts, stories, WhatsApp, email, follow-up sequences). Running all of them in a single synchronous request can hit serverless timeouts.

**MVP approach:**

- Create the `content_pack` row with status `processing`.
- Fan out asset-group generations with `Promise.all`.
- Persist assets as they land.
- Have the UI poll the pack until status is `ready`.

If this becomes fragile, the clean upgrade for a solo dev is a serverless job runner like Inngest or QStash, not a self-hosted Redis/worker setup. Only add a queue when you feel real pain.

---

## 2. Data Model Proposal (MVP)

Use multi-tenancy from day one via a **team** that every user belongs to (a solo agent just has a team of one). This costs almost nothing now and saves a painful migration later. Team UI arrives in Phase 3, but the schema supports it immediately.

### teams

- `id` (pk)
- `name`
- `plan` (enum: `trial` / `starter` / `pro` / `scale`)
- `stripe_customer_id`
- `stripe_subscription_id`
- `subscription_status`
- `current_period_end`
- `created_at`

### users

- `id` (pk)
- `email` (unique)
- `name`
- `image`
- `created_at`
- Plus Auth.js account/session/verification_token tables (Prisma adapter manages these).

### memberships

- `id` (pk)
- `user_id` (fk → `users`)
- `team_id` (fk → `teams`)
- `role` (enum: `owner` / `admin` / `member`)
- `created_at`
- Unique constraint on (`user_id`, `team_id`).

### agent_profiles

- `id` (pk)
- `team_id` (fk)
- `brand_name`
- `market` (e.g., "Dubai Marina")
- `tone_of_voice` (enum + free text)
- `languages` (text[])
- `platforms` (text[]: `instagram` / `facebook` / `linkedin` / `whatsapp` / `email`)
- `created_at`

### listings

- `id` (pk)
- `team_id` (fk)
- `created_by` (fk → `users`)
- `property_type`
- `area`
- `price`
- `currency`
- `beds`
- `baths`
- `size`
- `size_unit`
- `selling_points` (`jsonb` or `text[]`)
- `target_audience` (enum: `buyer` / `investor` / `tenant` / `seller`)
- `image_urls` (text[])
- `portal_url` (nullable — used later for auto-import in Phase 2)
- `status` (enum: `draft` / `active` / `archived`)
- `created_at`
- `updated_at`

### content_packs

- `id` (pk)
- `listing_id` (fk)
- `team_id` (fk)
- `status` (enum: `processing` / `ready` / `failed`)
- `schedule` (`jsonb` — the 7–14 day plan)
- `error` (nullable)
- `created_at`

> One listing can have several packs (regenerated over time).

### content_assets

- `id` (pk)
- `content_pack_id` (fk)
- `channel` (enum: `instagram` / `facebook` / `linkedin` / `whatsapp` / `email` / `story` / `reel`)
- `asset_type` (enum: `caption` / `reel_script` / `story_panel` / `whatsapp_broadcast` / `whatsapp_reply` / `email`)
- `variant_index` (int)
- `title` (nullable)
- `body` (text)
- `meta` (`jsonb` — e.g., hook, panel order, etc.)
- `created_at`

### follow_up_sequences

- `id` (pk)
- `content_pack_id` (fk)
- `audience_type` (enum: `buyer` / `investor` / `seller`)
- `created_at`

### follow_up_steps

- `id` (pk)
- `sequence_id` (fk → `follow_up_sequences`)
- `day_offset` (int)
- `channel` (`whatsapp` / `email`)
- `body` (text)
- `order` (int)

### usage_counters

- `id` (pk)
- `team_id` (fk)
- `period` (e.g., "2026-07")
- `listings_generated` (int)
- `regenerations` (int)
- Unique constraint on (`team_id`, `period`).

> This model cleanly enforces "10 listings/month" or similar limits without recomputing from Stripe.

### Key Relationships

- `team` 1—* `memberships` —1 `user`
- `team` 1—* `listings`
- `team` 1—1 `agent_profile` (MVP; later can be 1—* for multiple brand presets)
- `listing` 1—* `content_packs`
- `content_pack` 1—* `content_assets`
- `content_pack` 1—* `follow_up_sequences` 1—* `follow_up_steps`
- `team` 1—* `usage_counters`

---

## 3. Feature Breakdown by Phase

### Phase 1 — MVP

**Pages (Next.js App Router):**

- `/` — Minimal marketing/landing page.
- `/pricing` — Plan overview.
- `/login` — Auth entry point.
- `/onboarding` — Create team + agent profile.
- `/dashboard` — Listings/campaigns list + quota status.
- `/listings/new` — Create listing.
- `/listings/[id]` — View/edit listing.
- `/campaigns/[packId]` — Campaign overview: assets by channel + schedule + export.
- `/settings/profile` — Agent profile settings.
- `/settings/billing` — Billing and subscription management.

**API Endpoints (route handlers):**

- `GET/POST /api/auth/[...nextauth]` — Auth.js.
- `GET/POST /api/profile` — Read / upsert agent profile.
- `POST /api/listings` — Create listing.
- `GET /api/listings` — List current team's listings.
- `GET/PATCH/DELETE /api/listings/[id]` — Read / update / delete listing.
- `POST /api/listings/[id]/generate` — Kick off content pack generation (quota-checked) → returns `packId`, `status=processing`.
- `GET /api/content-packs/[id]` — Poll pack + assets + sequences + status.
- `POST /api/content-assets/[id]/regenerate` — Regenerate one asset (quota-limited).
- `GET /api/content-packs/[id]/export?format=md` — Export content pack as Markdown.
- `GET /api/usage` — Current period quota/usage.
- `POST /api/billing/checkout` — Stripe Checkout session.
- `POST /api/billing/portal` — Stripe Customer Portal session.
- `POST /api/webhooks/stripe` — Sync subscription state from Stripe.

**Golden-path user workflow:**

1. Sign in.
2. Onboarding creates team + membership + agent profile.
3. Go to `/listings/new`, submit a listing.
4. App redirects to `/campaigns/[packId]` showing a "generating" state.
5. UI polls `/api/content-packs/[id]` until `status=ready`.
6. User copies/exports assets and sees follow-up sequences displayed.
7. Quota decrements on generation; once exceeded, show upgrade prompt and block further generations.

### Phase 2 — Internet-Powered (Deferred)

- **Listing auto-import**
  - `POST /api/listings/import` (paste URL → adapter fetches/scrapes → prefilled draft).
  - Implement behind a `ListingImportAdapter` interface.
  - Respect target-site ToS; prefer official/partner APIs over scraping.

- **Area snapshots → content**
  - `market_snapshots` table and cached provider.
  - Auto-generate monthly market-update posts per area.

- **Up-and-coming discovery**
  - Hot-neighborhoods scoring, watchlist table.
  - Investor-content generation from watched properties.
  - Requires a scheduled refresh job.

- **Lead enrichment**
  - Personalize follow-ups with area/building data.

> Before Phase 2: research viable Dubai/UAE data sources (DLD open data, portal partner APIs, aggregators). Do not commit the architecture to a specific vendor until one is validated.

### Phase 3 — Analytics & Team/Brokerage

- Social performance ingestion + "what works for you" insights (requires OAuth to social platforms/schedulers).
- Team roles/permissions, shared brand templates, bulk generation, approve/edit flow.
- Direct social scheduling.
- Deeper CRM sync.
- Done-for-you marketplace integration.

---

## 4. File-Level Implementation Plan (Phase 1 Only)

```text
prisma/
  schema.prisma                      # all tables from §2
  migrations/                        # generated
lib/
  db.ts                              # Prisma client singleton
  auth.ts                            # Auth.js config (Google + email)
  stripe.ts                          # Stripe client + plan→price map
  quota.ts                           # check/increment usage_counters
  ai/
    client.ts                        # Claude wrapper: call, retry, JSON parse
    schema.ts                        # Zod schemas per asset type
    channelConstraints.ts            # length/format rules per channel
    generateContentPack.ts           # orchestrator (fan-out + persist)
    prompts/
      system.ts                      # shared system prompt (tone, facts)
      captions.ts
      reels.ts
      stories.ts
      whatsapp.ts
      email.ts
      followups.ts
      schedule.ts                    # 7–14 day plan generator
app/
  (marketing)/page.tsx               # landing
  pricing/page.tsx
  login/page.tsx
  onboarding/page.tsx
  dashboard/page.tsx
  listings/new/page.tsx
  listings/[id]/page.tsx
  campaigns/[packId]/page.tsx
  settings/profile/page.tsx
  settings/billing/page.tsx
  api/
    auth/[...nextauth]/route.ts
    profile/route.ts
    listings/route.ts
    listings/[id]/route.ts
    listings/[id]/generate/route.ts
    content-packs/[id]/route.ts
    content-packs/[id]/export/route.ts
    content-assets/[id]/regenerate/route.ts
    usage/route.ts
    billing/checkout/route.ts
    billing/portal/route.ts
    webhooks/stripe/route.ts
components/
  ListingForm.tsx
  CampaignBoard.tsx                  # assets grouped by channel
  AssetCard.tsx                      # copy + regenerate
  ScheduleTimeline.tsx
  FollowUpSequence.tsx
  QuotaBanner.tsx
  ProfileForm.tsx
middleware.ts                        # protect authed routes
```

**Supabase variation:**

- If you choose Supabase instead of Neon+Auth.js:
  - Replace `lib/auth.ts` (NextAuth setup) with Supabase Auth helpers.
  - Use Supabase Storage for `image_urls`/exports if needed.

---

## 5. Verification Checklist (Manual, End-to-End)

Use this checklist to manually test the MVP.

**Auth**

- Sign in with Google and with email magic link.
- Visiting `/dashboard` while logged out redirects to `/login`.

**Onboarding**

- New user is forced through `/onboarding`.
- Completing onboarding creates exactly:
  - One `team`.
  - One `membership` with role `owner`.
  - One `agent_profile`.

**Listing CRUD**

- Create a listing → it appears in `/dashboard`.
- Edit listing → changes persist.
- Delete listing → listing and its packs are removed.
- Attempt to access a listing from another team (by ID) → 403/404; multi-tenancy enforced.

**Generation**

- Click "Generate" on a listing.
- A `content_pack` row is created with status `processing`.
- UI polls `/api/content-packs/[id]`.
- Eventually `status` becomes `ready` and you see:
  - Captions (3–5 variants per selected platform).
  - Reel scripts.
  - Story panels.
  - WhatsApp + email templates.
  - Follow-up sequences with day offsets.
- Simulate network interruption mid-generate and confirm it ends as `failed` instead of stuck.

**Quota**

- On the Starter plan, the 11th generation in a month is blocked with an upgrade prompt.
- `usage_counters` increments correctly on each generation.
- Counters reset by period as designed.

**Regenerate**

- Regenerating one asset changes that asset only.
- Regeneration counts against a defined regeneration limit.

**Export**

- Markdown export contains all assets in a readable, sensible order.
- Per-asset copy-to-clipboard works.

**Billing**

- Checkout upgrades plan.
- Stripe webhook updates `plan` and `subscription_status` in DB (test with Stripe CLI `stripe listen`).
- Customer Portal cancellation downgrades access at period end as expected.

**Tenancy**

- Two different accounts never see each other's:
  - Listings.
  - Content packs.
  - Usage metrics.

---

## 6. Risks & Tradeoffs

- **LLM latency vs serverless timeouts**
  - Main technical risk.
  - Mitigation: fan-out generation per asset group + status polling, generous `maxDuration` on API routes.
  - If this becomes an issue, move to Inngest/QStash-level job orchestration. Do not build a custom queue prematurely.

- **AI cost & quality variance**
  - Constrain output length via `channelConstraints`.
  - Enforce JSON + Zod so malformed outputs fail loudly rather than silently.
  - Cap regenerations per plan to control cost.
  - Cache agent profile and system prompts to reduce token use.
  - Manually review sample packs after prompt changes.

- **PDF export scope creep**
  - Initial outputs are text and Markdown.
  - Avoid Puppeteer/Playwright PDF on serverless until users ask and you have bandwidth.

- **Over-building multi-tenancy**
  - Keep it simple: `teams` + `memberships` + row-level `team_id` filters.
  - No roles UI, no permissions matrix, no org-switcher in MVP.

- **Stripe webhook reliability**
  - Treat webhook as the source of truth for subscription state.
  - Make it idempotent.
  - Verify signatures.
  - Do not gate features solely on the client-side checkout redirect.

- **Phase 2 scraping legality/fragility**
  - Auto-import is the riskiest external dependency.
  - Isolate it behind an adapter interface.
  - Prefer official/partner APIs.
  - Never let it block manual listing intake.

- **Prompt injection (Phase 2)**
  - Once ingesting listing URLs/scraped text, treat that text as untrusted data, not instructions, in prompts.

- **Scope discipline**
  - The spec is rich; it is tempting to build discovery features early.
  - Phase 1 is the critical loop: listing in → pack out → export → monetized.
  - Nothing else earns its place until that loop is being used repeatedly in the real world.

---

## 7. Review & Self-Critique Loops

To improve quality, this project uses an explicit two-role loop for important plans and changes: a **Builder** role and a **Reviewer** role.

### 7.1 Builder Role Instructions (Plan Mode)

Use these whenever Claude is drafting a plan, spec, or non-trivial change:

> You are the **Builder**.  
> Your job is to produce the best possible plan, spec, or code given my request.  
> - Focus on clarity and explicit reasoning.  
> - Write as if another engineer will review and try to break it.  
> After youre done, stop. Do not self-review; a separate Reviewer pass will handle that.

Typical use:

1. Enter Plan Mode.
2. Tell Claude to read the relevant spec file (for example, this implementation plan or a feature-specific doc).
3. Ask the Builder to design or update a specific part of the system (e.g., "design the Phase 1 content generation pipeline"), following the instructions above.

### 7.2 Reviewer Role Instructions (Plan Mode)

Run this as a separate Plan Mode step, ideally in a fresh session or after loading only the artifact to review (plan, code, or spec):

> You are the **Reviewer**. Your job is to find gaps, risks, and incorrect assumptions — not to confirm the approach.  
> 1. Read the **entire** document or diff before commenting.  
> 2. For each finding, state:  
>    - What the artifact claims.  
>    - What is actually true or unclear.  
>    - A **specific fix** or improvement.  
> 3. Group findings by severity:  
>    - **Critical:** security, data loss, major correctness issues.  
>    - **Medium:** performance, maintainability, confusing UX, likely failure modes.  
>    - **Low:** naming, style, minor nits.  
> 4. If the artifact makes a claim about how a tool, API, or framework works, **verify it yourself** (using your own knowledge or docs) instead of trusting the claim.  
> 5. If you find zero issues, explicitly state that you attempted to break every assumption and still found no problems.

Reviewer workflow:

1. Point the Reviewer at a specific artifact (e.g., `docs/real-estate-content-engine.md` or `plans/feature-x.md`).
2. Ask for a structured list of findings grouped by severity.
3. Do not let the Reviewer rewrite the artifact; its job is only critique and suggested fixes.

### 7.3 Builder Incorporates Reviewer Feedback

After running the Reviewer pass, go back to the Builder instructions and ask Claude to integrate the feedback:

> As the **Builder**, update the plan/spec/code to incorporate all **Critical** and **Medium** findings from the Reviewer.  
> - Apply the fixes and improvements.  
> - If you disagree with any finding, explain why in 1–2 sentences, then either adjust or justify.  
> - Produce a clean, updated artifact with all changes applied, replacing the prior version.

You can repeat the Builder → Reviewer → Builder cycle once more for especially important artifacts (auth, billing, AI orchestration), but generally **one review loop** is enough for most changes.

### 7.4 Optional: Lightweight Self-Review Rubric

For smaller tasks where a full two-role loop is overkill, add this self-review step to individual prompts:

> Before returning your final answer, run a quick self-review:  
> 1. Evaluate your draft against this rubric, scoring each item from 1–5:  
>    - Correctness: does it do what the spec asks?  
>    - Completeness: does it cover all required pieces?  
>    - Safety/Robustness: any obvious failure cases or data issues?  
>    - Clarity: is the structure and naming clear?  
> 2. If any dimension is ≤3, write a short private critique and then revise the draft once to address those issues.  
> 3. Return only the revised draft.

This keeps quality high without infinite loops: at most **one revise pass** per small task, with the heavier two-role loop reserved for high-impact changes.
