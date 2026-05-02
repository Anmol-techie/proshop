# AI Visibility Repairs (`/actions`) — Full System Documentation

> One-stop reference for everything that runs behind the Actions page: the UI surfaces, the action-generation pipeline, the agentic content orchestrator, the content writer, and every metric / KPI shown on the page.
>
> Audience: engineers + analysts who need to reason about why an action appeared, why a draft says what it says, or why a number on the strip moved.
>
> Conventions in this doc:
> - File paths are repo-relative; line numbers (`file:NNN`) point to the current `beacon-redesign` branch.
> - "Engine A" = the legacy per-prompt action set; "Engine B" = `action-intelligence.service.ts` (the lane-based engine that does most of the work today).
> - Numbers in **bold** are magic constants you'll find verbatim in code — change them with care.

---

## Table of contents

0. [Glossary — every term used in this doc](#0-glossary--every-term-used-in-this-doc)
1. [Quick mental model](#1-quick-mental-model)
2. [Frontend — page anatomy and state](#2-frontend--page-anatomy-and-state)
3. [How actions are generated (Engine B)](#3-how-actions-are-generated-engine-b)
4. [Persistence: from `ActionSeed` to `Action` row](#4-persistence-from-actionseed-to-action-row)
5. [The orchestrator and its sub-agents](#5-the-orchestrator-and-its-sub-agents)
6. [Proposal validation](#6-proposal-validation)
7. [Content drafting (per play type)](#7-content-drafting-per-play-type)
8. [Lifecycle, events, and the closed-loop outcome worker](#8-lifecycle-events-and-the-closed-loop-outcome-worker)
9. [Metrics, KPIs, and how they're computed](#9-metrics-kpis-and-how-theyre-computed)
10. [API endpoints called by the page](#10-api-endpoints-called-by-the-page)
11. [Knobs, env vars, and queue topology](#11-knobs-env-vars-and-queue-topology)
12. [Failure modes and where to look first](#12-failure-modes-and-where-to-look-first)

---

## 0. Glossary — every term used in this doc

> If a term in any later section is unfamiliar, look it up here first. Definitions are written so a reader who has never opened the codebase can follow the rest of the doc cold. Within a definition, *italicised* terms are themselves defined elsewhere in this glossary.

### 0.1 Domain entities (database tables and conceptual nouns)

| Term | What it is |
|---|---|
| **Company** | A tenant/brand the user is monitoring. Owned by an `Organization`. Has many *Prompts*, *Competitors*, *Simulations*, *Citations*, etc. |
| **Competitor** | A rival brand the user has explicitly told us to track for this company. Stored in `Competitor` table; also derived heuristically from `Simulation.mentionsCompetitors`. |
| **Prompt** | A natural-language question we run against the AI models on the company's behalf (e.g. "Top japanese fine dining restaurants in Dubai"). Prompts have a *category* (discovery, comparison, trust_validation), a *topic*, and an `isActive` flag. Soft-deleted prompts have `isActive=false` and are excluded by every per-project page (see §9). |
| **Simulation** | One model's response to one prompt at one moment in time. Carries `responseText`, `mentionsCompany` (boolean), `mentionsCompetitors` (JSON array of competitor mention shapes), `presenceQuality`, `recommendationPosition`, `sentiment`, `aiProvider`, `aiModel`, etc. The atom Engine B feeds on. |
| **SimulationRun** | A scheduled batch that produced N simulations across multiple prompts/models in one shot. |
| **Citation** | One URL the AI cited inside a single simulation's response. Joins back to a `CitationSource`. |
| **CitationSource** | A unique URL the AI has ever cited for this company. Carries `domain`, `url`, `trustTier`, `mentionStatus` (whether the company appears on the page), `mentionReason` (why mention failed: `http_403`, `http_404`, etc.), and a denormalised `citationCount`. |
| **Action** | A unit of recommended work (e.g. "Pitch a guest post to Forbes"). Persisted in `Action` table. Carries `priority`, `title`, `reason`, `impactArea`, `evidenceType`, `metadata` (JSON catch-all), `status` (open/completed/dismissed/in_progress). |
| **ActionSeed** | An in-memory action object produced by Engine B before it gets persisted into an `Action` row. Same fields as `Action` but no `id`/`createdAt`. The dedupe + cap step in `regenerateActionsForCompany` decides which seeds become rows. |
| **ActionEvent** | A row in the funnel-tracking table for actions: `kind ∈ {opened, plan_started, plan_completed, draft_viewed, draft_copied, draft_approved, draft_rejected, action_completed, action_dismissed}`. Best-effort writes; never throws. Used by Phase 6 to compute conversion gradients. |
| **ActionOutcome** | A row recording the measured effect of completing an action: visibility/sentiment baselines vs. +14d/+30d windows, count of new citations on target domains, count of concurrent overlapping actions. Filled by the *outcome worker* nightly. |
| **Boost** | A user-customised "workflow" (e.g. multi-step playbook). May or may not be tied to an Action. Has its own steps, status (`DRAFT`, `RUNNING`, `AWAITING_APPROVAL`, `PAUSED`, `COMPLETED`, `FAILED`), and progress %. *Orphan boosts* (no `actionId`) appear as first-class items in the queue. |
| **OrchestratorRun** | One end-to-end run of the agentic content orchestrator. Carries `status (PENDING→PLANNING→EXECUTING→COMPLETED/FAILED/CANCELLED)`, `trigger (MANUAL/POST_SIMULATION/CRON)`, `plan` (JSON proposals), `summary`, `tokensUsed`, `costUsd`, `errorMessage`, timing. |
| **OrchestratorStep** | One step inside a run: a single tool call, sub-agent invocation, or critique pass. Step `kind ∈ {PLAN, TOOL_CALL, SUB_AGENT, CRITIQUE, OBSERVATION, RETRY, FINAL}`. Powers the live tool-call indicator and trajectory drawer in the UI. |
| **OrchestratorLearning** | A distilled "lesson" the orchestrator wrote to itself for next time, e.g. "TripAdvisor pitch via web form has 0% acceptance — skip." Read via `get_learnings` tool. |
| **ContentDraft** | The persisted output of one `generate_play` call: a draft `{title, slug, contentType, playType, status='ready', markdown, html, jsonLd, metaTags, instructions}`. Linked back to the originating `actionId`. |
| **DailySnapshot** | A per-day, per-company aggregation of visibility / sentiment / SoV metrics. Used by the *outcome worker* as the baseline / +14d / +30d data source. |
| **DomainOverride** | An admin-set channel-classification override (force `g2.com → REVIEW_PLATFORM` etc.). Loaded once at the start of `generateIntelligentActions` and used by the channel classifier. |

### 0.2 Engine concepts

| Term | What it is |
|---|---|
| **Engine A** | The legacy per-prompt action set (`generatePerPromptActions`) — one action per losing prompt. Still alive but mostly superseded; its output is merged with Engine B's inside `regenerateActionsForCompany`. |
| **Engine B** | `action-intelligence.service.ts` — the lane-based engine described in §3. Mines simulations + citations to produce thematic, deduped, prioritised actions. Produces ~80% of what's in the queue today. |
| **Lane** | One of the seven analytical pipelines inside Engine B (prompt-gap, sentiment, citation-DNA, channel, model-divergence, surface-scout, hub/topic). Each runs independently on the same pre-loaded dataset and contributes `ActionSeed`s. |
| **`generateIntelligentActions(companyId)`** | The Engine B entry point. Loads sims/citations/sources in parallel, runs all 7 lanes, sorts/dedupes/caps, returns `ActionSeed[]`. |
| **`regenerateActionsForCompany(companyId)`** | The route-level wrapper: calls Engine A + Engine B, applies the 3-layer regenerator dedup, caps `citation_dna` at 8, persists new rows, touches existing rows. Triggered by the "Scan for actions" CTA and by the staleness check on `GET /actions`. |
| **dedupeHash** | `SHA256(impactArea | evidenceType | playType | sorted(domains) | sorted(promptIds)).slice(0,32)` — a deterministic per-seed fingerprint so the same evidence never spawns two open rows across runs. |
| **evidenceType** | Tag identifying which lane produced an action: `prompt_gap`, `objection`, `citation_dna`, `model_divergence`, `surface_scout`, `hub_gap`, `channel_strategy`, `manual`. Used in the regenerator dedup and by Phase 7 conversion analysis. |
| **impactArea** | The kind of metric an action moves: `visibility`, `citations`, `sentiment`, `general`. Drives KPI strip aggregation and queue stage bucketing. |
| **playType** | The kind of content an action ultimately produces — `blog_post`, `linkedin_post`, `reddit_response`, `email_pitch`, `comparison_article`, `directory_submit`, `marketplace_claim`, `wiki_edit`, `award_submit`, `review_solicit`, `b2b_outreach`, `on_domain_content`, `community_post`, `forum_response`, `quora_answer`, `twitter_thread`, `youtube_video`, `tiktok_video`, `instagram_post`, `social_post`, `surface_scout`. Drives both `proposal-validator.ts` validity check and `play-cards.tsx` per-platform rendering. |
| **Per-prompt vs per-cluster scoring** | Engine B first computes a `weightedRisk` for each *individual prompt* (formula in §3.4), then aggregates those into a *cluster* score (`revenueScore = round(Σ weightedRisk × 10)`) at the topic or competitor level. Priority is decided on the cluster, not the prompt. |
| **Touch-only update** | The third pass of the regenerator: rather than re-`update()` every existing matched action (which exhausted Postgres connections on big tenants), it bumps `updatedAt` and merges metadata via raw SQL `JSONB_MERGE`. |

### 0.3 Source classification

| Term | What it is |
|---|---|
| **trustTier** | Per-CitationSource authority bucket: `TIER_1` (mainstream press / wikipedia / .gov / .edu), `TIER_2` (industry pubs, established reviews), `TIER_3` (long-tail blogs, small forums). Set by the citation pipeline upstream of Engine B. Used as a multiplier in Engine B's scoring. |
| **trustWeight** | Numeric multiplier applied to `trustTier` inside scoring formulas. Engine B citation-DNA lane uses `{TIER_1:3, TIER_2:2, TIER_3:1}` (`:1211`); the surface-scout lane uses a *steeper* `{TIER_1:4, TIER_2:2, TIER_3:1}` (`:886`) so a single TIER_1 long-tail page can outrank dozens of TIER_3 ones. |
| **channelKey** | Outreach channel a citation source belongs to. 16 buckets (see §3.6 for the table). Computed deterministically by `lib/channel-classifier.ts` from the URL/domain (with `DomainOverride` as the manual override). |
| **CHANNEL_META** | Per-channel metadata struct: `{ key, label, description, userAction, priority }`. `userAction` is the one-line "how to act on this channel" string surfaced as `metadata.channelAction` on actions. |
| **mentionStatus** | Per-CitationSource flag set by the page-fetcher: `MENTIONED`, `UNDETECTED` (we fetched the page; the brand isn't there), or unset. Used in the catch-all citation_dna emitter to add an "Your brand is not even mentioned on this page" line. |
| **mentionReason** | If the fetcher couldn't decide whether the brand is mentioned, this carries the failure reason: `http_403`, `http_404`, `http_5xx`, `fetch_failed`. Sources with these reasons are filtered out (see `UNREACHABLE_REASONS` below) unless they're competitors. |
| **presenceQuality** | Per-Simulation classification of *how* the brand appears in a model response: `PRIMARY` (the recommendation), `SECONDARY` (one of a small shortlist), `MENTION` (just listed), `ABSENT`. Numeric weights in `COMPETITOR_PRESENCE_SCORE`. |
| **competitorHubCount** | For one source, how many *distinct* tracked competitors of ours have ever been cited on it. ≥2 means it's a "competitor hub" — they share authority on this domain. |
| **companyMentioned / companyAbsent / companyPrimary** | Per-CitationSource counters: number of cited simulations where the brand was, respectively, mentioned at all / completely absent / the PRIMARY recommendation. |

### 0.4 Scores and metrics (alphabetical)

| Term | Definition |
|---|---|
| **channelOpportunityScore** | `round(Σ sourceOpportunityScore for sources in the channel × 10)`. Used to rank channel-strategy buckets (`:1388`). |
| **competitorHubCount** | See "Source classification" above. |
| **consensusScore** | `mentionedModels.length / totalModels` for one prompt in the model-divergence lane. Used as the secondary HIGH gate (`≥0.8` for visibility, `≥0.75` for sentiment) when the count of affected gaps is below 4. |
| **dropScore** | Momentum lane only. `(previousMentionRate − recentMentionRate) × trustWeight × citationCount`. The bigger this number, the steeper the recent ranking regression on that source. |
| **frequency** | Objection-mining lane only. `uniqueSimsWithTheme / totalMentionedSims × 100`. Drives the HIGH/MEDIUM/LOW ladder for objection actions. |
| **hubScore** | `trustWeight × citationCount × max(1, competitorHubCount)`. Used inside hubInfoBySourceId to rank competitor citation hubs. |
| **lossRate** | `companyAbsent / totalSims` for one source. The fraction of cited answers where this source surfaced but you didn't. Used as a multiplier in `sourceOpportunityScore`. |
| **lossRatePct** | `round(lossRate × 100)`. Stamped onto `targetDomains[].lossRatePct` for UI display. |
| **opportunityScore (bucket)** | `Σ sourceOpportunityScore` over a channel's sources. Used to pick the top-2 channel buckets. |
| **opportunityScore (source)** | `trustWeight × max(1, citationCount) × max(0.35, lossRate) + competitorHubCount × 3 + (momentum ? 6 : 0)` (`:1331`). The single per-source ranking number used everywhere in citation-DNA. |
| **primaryLossRate** | `topCompetitor.primaryWins / totalRuns` — the share of runs in which the strongest competitor was the *primary* recommendation. Secondary HIGH trigger in Lane 1 (≥0.6 discovery, ≥0.55 comparison). |
| **primaryWins** | Per-competitor-per-prompt counter: how many simulations had this competitor in `presenceQuality=PRIMARY` or `position=1`. |
| **revenueScore** | Lane 1 cluster score: `round(Σ weightedRisk × 10)` over a topic/competitor cluster. The "is this discovery topic a HIGH?" trigger when ≥45 (discovery) or ≥40 (comparison). Phase 4.1 renamed this to `riskScore` — the codebase still writes both for backward compat. |
| **riskScore** | Same value as `revenueScore`. Reads from `metadata.riskScore` first, falls back to `metadata.revenueScore` for old rows. The honest-naming alias because the score isn't actually dollars (see §9.5). |
| **scoreThreshold / rateThreshold** | The numbers fed to `explainPriority()` so the UI can render "HIGH because riskScore 50 ≥ 45 AND primaryLossRate 0.62 ≥ 0.6" tooltips. |
| **sourceOpportunityScore** | See "opportunityScore (source)" above. |
| **strength (`getActionStrength`)** | Composite tiebreaker score used during the dedupe sort — see §3.11.8 for the formula. The action with the highest strength survives when two emitters target the same domain/title. |
| **weightedRisk** | Per-prompt risk score in Lane 1: `lossRuns × CATEGORY_WEIGHTS[category] + primaryWins × 1.6 + max(0, competitors-1) × 0.35`. Aggregated up to `revenueScore`. |

### 0.5 Constants and pattern sets

| Term | Value (file:line) | Purpose |
|---|---|---|
| **CATEGORY_WEIGHTS** | `{discovery:1.0, comparison:1.2, trust_validation:1.5, other:1.0}` (`:156`) | Funnel-stage urgency multiplier in Lane 1. |
| **COMPETITOR_PRESENCE_SCORE** | `{PRIMARY:1.0, SECONDARY:0.7, MENTION:0.35, ABSENT:0}` (`:170`) | Numeric weight per `presenceQuality`. |
| **UNREACHABLE_REASONS** | `{http_403, http_404, http_5xx, fetch_failed}` (`:117`) | Citation sources we can't pitch (page won't load); skipped unless channel is `COMPETITOR` (where we make defensive on-domain content anyway). |
| **HEDGING_PATTERNS** | 9 regex patterns (`:121–131`) | Detects "however", "some users report", "drawback", "mixed reviews", etc. in AI responses. The trigger for objection mining. |
| **THEME_PATTERNS** | 8 regex patterns (`:209–218`): pricing, quality, service, features, experience, reliability, location, competition | Classifies a hedged sentence into a theme bucket. |
| **AI_TELL_PHRASES** | Find/replace dictionary (`content-lab.service.ts`) | Strips telltale AI phrasing from `reddit_response` drafts. |
| **META_LEAK_PATTERNS** | Regex set (`content-lab.service.ts`) | Strips lines that talk about AI visibility / rankings / SEO from generated content. |
| **ANCHOR_COMMUNITY_DOMAINS** | `{reddit.com, quora.com, linkedin.com, x.com, twitter.com, medium.com}` (`proposal-validator.ts`) | Domains that count as "anchored" for surface-scout proposals; anything else gets rejected. |
| **PITCH_PLAY_TYPES** | 7 types: `email_pitch, guest_post_pitch, listicle_inclusion_pitch, product_listing_request, partnership_inquiry, editorial_tip, press_release` (`:37–45`) | The play types that go through the LLM domain critic in pass 8 of validation. |
| **VALID_PLAY_TYPES** | 26 strings (`proposal-validator.ts:49–71`) | Whitelist; proposals with any other `playType` are rejected at pass 3. |
| **PLAY_INSTRUCTIONS** | Per-`playType` system-prompt template (`content-lab.service.ts:373+`) | The actual writing brief Sonnet receives for each draft. |
| **CRITIC_THRESHOLD** | `0.6` (`critic.ts`) | Drafts with score < 0.6 get rejected; the writer retries once. |
| **PER_ACTION_CAP** | `8` (`proposal-validator.ts:74`) | Max proposals per actionId, applied at validation pass 4. |
| **SURFACE_SCOUT_PER_COMPANY_CAP** | `15` (`surface-scout.ts:39`) | Max long-tail domains scouted per Engine B run. |
| **ABSENT_RATIO_FLOOR / ABSENT_ABSOLUTE_FLOOR** | `0.3 / 2` (`:911-912`) | Surface-scout weak-mention gates: domain qualifies if absent ≥30% of cites OR ≥2 absolute absences. |
| **CITATION_DNA_CAP** | `8` (regenerator) | Phase 2.2 cap on how many `citation_dna` actions can be open at once (citation-DNA was 48% of the queue and underperformed on conversion). |
| **MAX_TURNS** | `10` (`visibility-orchestrator.ts:37`) | Hard cap on tool-use turns in a plan run. Real runs converge in 3–4. |
| **RUN_TIMEOUT_MS** | `180_000` (`:42`) | Wall-clock cap per orchestrator plan run. |
| **DEFAULT_DAILY_TOKEN_CAP** | `2_000_000` (`budget-guard.ts`) | Per-tenant per-day Anthropic token cap. |
| **DEFAULT_DAILY_USD_CAP** | `5.00` (`budget-guard.ts`) | Same in dollars; whichever cap hits first kills the run. |
| **DAILY_CRON** | `"0 6 * * *"` (`orchestrator.worker.ts`) | Autopilot fires at 06:00 UTC if `AUTOPILOT_CRON_ENABLED=true`. |
| **REAP_THRESHOLD_MS** | `420_000` (`orchestrator-reaper.ts`) | `2 × RUN_TIMEOUT + 60s safety`. Runs older than this with non-terminal status get force-FAILED. |

### 0.6 Sub-agents (LLM and non-LLM)

| Sub-agent | File | Model | One-line job |
|---|---|---|---|
| **research** | `agents/sub-agents/research.ts` | none (pure SQL) | Look up actions, prompts, sims, citations, competitors, past plays. The orchestrator's eyes into the database. |
| **planner** | `agents/sub-agents/planner.ts` | none (delegates) | Wraps `contentLabService.suggestPlays()` to return ranked plays for one action. |
| **content-writer** | `agents/sub-agents/content-writer.ts` | calls Sonnet via content-lab | Adapter that calls `contentLabService.generateDraft()` with action context. Preserves the original LinkedIn/Reddit/Twitter/Quora prompts byte-for-byte. |
| **critic** | `agents/sub-agents/critic.ts` | `claude-sonnet-4-6` | Judges every freshly-written draft on evidence alignment / platform fit / concrete facts / citation-worthiness / brand safety. Returns `{score, accept, reasons}`. |
| **surface-scout** | `agents/sub-agents/surface-scout.ts` | Haiku (summary) + Sonnet (3 candidates) | The only LLM-driven action *generator*. Fetches an UNCATEGORIZED long-tail page, summarises it, and produces 3 ranked next-best actions a user could take. |
| **proposal-validator** | `agents/orchestrator/proposal-validator.ts` | Haiku (domain critic only) | Eight-pass filter applied to every `propose_plays` call: validity, cap, competitor blocklist, dedupe, channel enrichment, LLM domain classifier, scout anchoring. |

### 0.7 Tools the orchestrator can call

| Tool | One-line definition |
|---|---|
| **list_actions** | Return open Actions for the company. Capped at 30, sorted HIGH→MED→LOW. |
| **get_evidence** | Fetch one Action with affected prompts, simulation excerpts, per-prompt citation sources, and (if surface_scout) the scout candidates/summary/domain. |
| **get_citations** | Return the company's citation sources. Optional filters: `minCount`, `unmentionedOnly`, `bucket`. |
| **get_competitor_presence** | Return `[{domain, competitors:[]}]` — which tracked rivals appear on each cited domain. |
| **get_past_plays** | Return up to 20 recent `ContentDraft` rows so the model can avoid re-writing the same angle. |
| **get_learnings** | Return up to 10 most recent `OrchestratorLearning` rows for the company. |
| **propose_plays** | Submit a list of proposals. Plan-mode only. Triggers the validator + critic. |
| **generate_play** | Create one ContentDraft via writer + critic loop. **Disabled in plan mode** — only fires in execute mode. |
| **write_memory** | Persist a lesson into `OrchestratorLearning` for future runs. |
| **finish** | End the loop and emit a final summary. |

### 0.8 Orchestrator runtime concepts

| Term | What it is |
|---|---|
| **plan mode** | First half of the agentic flow: orchestrator chooses which actions to act on and what proposals to make. Output: `OrchestratorRun.plan` (JSON proposals). |
| **execute mode** | Second half: user picks proposals → orchestrator runs `generate_play` for each → drafts get persisted. Output: `ContentDraft` rows. |
| **focusActionId** | Optional input param: scope a plan run to one action's evidence so it doesn't pull the whole catalog (saves tokens). Set automatically by the auto-trigger when the user opens a specific action. |
| **6h auto-trigger cooldown** | The page won't fire a fresh plan run for the same company more than once per 6 hours, even if the user re-opens actions. Stamped on `OrchestratorRun.metadata.lastTriggerAt`. |
| **trajectory logger** | The thing that writes `OrchestratorStep` rows. Powers the live tool-call indicator and the trajectory drawer in MissionControl. |
| **budget guard** | Enforces per-tenant daily token + USD caps before a run starts. |
| **reaper** | Background sweep (every 60s) that marks runs as `FAILED` if they've been non-terminal longer than `REAP_THRESHOLD_MS`. Doesn't cancel the in-flight Anthropic call — just marks the row terminal so the UI doesn't lie. |
| **BullMQ** | The Redis-backed job queue. The orchestrator never runs in-request — every `POST /orchestrate` enqueues a job onto `orchestratorQueue` (worker concurrency 4). |
| **autopilot cron** | A daily BullMQ repeatable job that fans out one plan run per active company. Disabled by default; toggle via `AUTOPILOT_CRON_ENABLED=true`. |
| **runWithUsageContext** | Worker wrapper that scopes all nested LLM token usage to `(companyId, feature)` so per-tenant cost reporting works. |

### 0.9 Frontend concepts

| Term | What it is |
|---|---|
| **`?sel=` URL param** | URL-synced selection key on the actions page. Encoded as `action:{id}` / `boost:{id}` / `pitch:{domain}`. |
| **PRIORITY_RANK** | `{HIGH:0, MEDIUM:1, LOW:2}` — sort key used by the "Recommended next" banner. |
| **navOrderActions** | Frontend-derived sort by *queue stage* (then priority/createdAt) used for `j`/`k` keyboard navigation. **Not** the same order as the recommended-next banner — that's a known ergonomic quirk. |
| **outcome stage** | One of 9 buckets the queue groups actions into (Needs your input / Get mentioned in editorial / Get more reviews / Win "best of" searches / Fix our website for AI / Running / Completed / Failed / Dismissed). Derived from `status / impactArea / channelKey / playType`. |
| **MissionControl** | The pill at the top of the page showing in-flight orchestrator status with a pulsing dot. Click "Run now" to open the pre-flight modal. |
| **PlayDraftRenderer** | Frontend component that routes a `ContentDraft` to one of ~8 per-platform renderers based on `playType`. |
| **action-events** | The `frontend/lib/action-events.ts` helper. Fire-and-forget POST to `/beacon/company/:id/action-event` for every funnel kind. |
| **timeRange filter** | `7 / 30 / 90 / latest` (Zustand store, persisted to localStorage). Appended to the actions GET as a query param. |
| **model filter** | `all / openai / gemini` filter on which AI provider's simulations to count. Same store. |
| **selectedDate** | Optional historical scrubber date (YYYY-MM-DD). When set, the page reads metrics from that day's `DailySnapshot` instead of "now". |

### 0.10 Models, providers, and external services

| Term | What it is |
|---|---|
| **Sonnet** | `claude-sonnet-4-6` — the orchestrator's planning model and the writer/critic's judgement model. |
| **Haiku** | `claude-haiku-4-5-20251001` — the cheaper/faster model used for surface-scout page summaries and the proposal validator's domain classifier. |
| **AI provider / aiModel** | `Simulation.aiProvider` (e.g. `openai`, `anthropic`, `google`) and `aiModel` (e.g. `gpt-4.1-2025-04-14`). What ran a given simulation. |
| **getPlatformLabel** | Frontend-friendly label for a `(provider, model)` pair (e.g. `"ChatGPT"`, `"Gemini"`, `"Claude"`, `"Perplexity"`). Used in divergence-action titles. |
| **classifyDomain** | Deterministic URL → `channelKey` classifier (`lib/channel-classifier.ts`). Combines string rules + the `DomainOverride` admin overrides. |
| **filterCompetitorDomains** | Haiku-backed safety filter that takes a list of domains and partitions them into `{safe, competitor}`. Used inside Lane 3 to keep us from pitching a competitor. |

### 0.11 Concepts mentioned only in passing

| Term | What it is |
|---|---|
| **Phase N** | Internal milestone naming convention used in commits and code comments. *Phase 1.1* = ActionEvent funnel; *Phase 2.2* = citation_dna cap; *Phase 3* = ActionOutcome; *Phase 4.x* = dedupe/regenerator hardening; *Phase 5.1* = surface scout; *Phase 6* = re-fitting magic weights from outcome data; *Phase 7* = conversion-gradient analysis; *Phase 8* = `priorityExplanation` tooltip. |
| **Beacon** | Internal codename for the public AI-visibility dashboard the `/companies/[id]/(beacon)/...` routes live under. |
| **Velova** | Public marketing brand for the same product. |
| **GEO** | "Generative Engine Optimisation" — the discipline of making a brand show up well in LLM answers. The reason this whole system exists. |

---

## 1. Quick mental model

```
                ┌─────────────────────┐
 Simulations ──▶│ action-intelligence │── ActionSeed[] ──┐
 Citations  ──▶ │  service (Engine B) │                  │
 Sources    ──▶ │  + per-prompt set   │                  ▼
                │  (Engine A legacy)  │           ┌─────────────┐
                └─────────────────────┘           │ regenerate  │
                                                  │ ActionsFor  │── Action rows
                                                  │  Company()  │ (Prisma)
                                                  └──────┬──────┘
                                                         │
                                                         ▼
                                                 ┌──────────────┐
   User opens action  ───────────────────▶       │ /actions UI  │
   (auto-trigger run, 6h cooldown)               │  (queue +    │
                                                  │   detail)   │
                                                  └─────┬────────┘
                                                        │
                                          POST /orchestrate (plan)
                                                        ▼
                                              ┌──────────────────┐
                                              │ orchestrator.worker  │
                                              │ (BullMQ, conc=4) │
                                              └─────┬────────────┘
                                                    │
                                                    ▼
                                            visibility-orchestrator (Sonnet)
                                            ├─ list_actions
                                            ├─ get_evidence
                                            ├─ get_citations
                                            ├─ get_competitor_presence
                                            ├─ get_past_plays / get_learnings
                                            ├─ propose_plays   ◀── proposal-validator (8 passes)
                                            └─ finish
                                                    │
                                            User picks proposals → "Generate"
                                                    ▼
                                            execute mode → handlers.generate_play
                                                    ├─ contentWriter.write()  → contentLab.generateDraft (Sonnet, per-platform prompts)
                                                    └─ critic.evaluate() (Sonnet, score ≥ 0.6 to keep)
                                                          ▼
                                                    ContentDraft rows + ActionEvent funnel
```

The page itself never calls Anthropic directly. Every LLM call is queued through BullMQ → `orchestrator.worker.ts` → `visibility-orchestrator.ts`.

---

## 2. Frontend — page anatomy and state

### 2.1 Layout

**Route file:** `frontend/app/companies/[id]/(beacon)/actions/page.tsx` (~1077 lines, component `CommandCenterPage`).

```
┌─────────────────────────────────────────────────────────────┐
│ ActionsTabBar                                               │  sub-nav
├─────────────────────────────────────────────────────────────┤
│ Hero ("AI Visibility Repairs", open-issue line, CTAs)       │
├─────────────────────────────────────────────────────────────┤
│ Custom-goal input (collapsible)                             │
├─────────────────────────────────────────────────────────────┤
│ MissionControl pill (idle / queued / planning / running)    │
├─────────────────────────────────────────────────────────────┤
│ PageSummaryStrip (LLM-generated narrative)                  │
├─────────────────────────────────────────────────────────────┤
│ SummaryStrip — 4 KPI cards (§9.2)                           │
├─────────────────────────────────────────────────────────────┤
│ Recommended-next banner (top open HIGH action)              │
├─────────────────────────────────────────────────────────────┤
│ Two-pane shell (xl ≥ 1280px):                               │
│   ┌────────────────────────┬───────────────────────────┐    │
│   │ ActionQueue (440px)    │ ActionDetailPane / Boost  │    │
│   │ - search + j/k nav     │   / Pitch detail pane     │    │
│   │ - 9 outcome stages     │ - evidence prompts        │    │
│   │ - first-class:         │ - proposals + drafts      │    │
│   │   custom workflows     │ - "where to deploy" CTAs  │    │
│   │   pitch outlets        │                            │    │
│   └────────────────────────┴───────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
Below xl: single column. Detail opens in `RightDrawer` overlay.
```

Two-pane container is sticky-bounded so each pane scrolls independently (commit `88f2a79`). Below `xl` the drawer mounts only on demand to avoid the body-overflow leak we hit at the breakpoint switch.

### 2.2 URL-synced selection (`?sel=`)

Selection is encoded as a single `SelectedKey` string:

| Form          | Meaning                                  |
|---------------|-------------------------------------------|
| `action:{id}` | One Action row from the queue            |
| `boost:{id}`  | An orphan custom workflow (no actionId)  |
| `pitch:{domain}` | A recommended outreach outlet         |

Encoder/decoder lives in `action-queue.tsx:33–46` (`selKey.action()`, `selKey.boost()`, `selKey.pitch()`, `selKey.parse()`). `setSelectedKey()` (`page.tsx:57–68`) calls `router.replace()` without scroll and fires the `opened` event the first time per page-load (deduped via `openedFiredRef`).

If the entity behind a selection disappears (action dismissed, boost deleted), the cleanup effect at `page.tsx:433–442` clears `?sel=`.

### 2.3 ActionQueue — 9 outcome stages

`frontend/components/beacon/action-queue.tsx:71–81` defines the stage list. Each action is bucketed by `status / impactArea / channelKey / playType`:

| # | Stage                         | Trigger rule                                                   |
|---|-------------------------------|----------------------------------------------------------------|
| 1 | Needs your input              | Boost has a step in `AWAITING_APPROVAL`                        |
| 2 | Get mentioned in editorial    | impactArea=`visibility \| citations` + editorial-ish playType  |
| 3 | Get more reviews              | impactArea=`sentiment` or channelKey=`REVIEWS` or `/review/`   |
| 4 | Win "best of" searches        | comparison/best-of plays                                       |
| 5 | Fix our website for AI        | on-site / website / tech plays                                 |
| 6 | Running                       | Associated boost has status=`RUNNING`                          |
| 7 | Completed                     | status=`completed`                                             |
| 8 | Failed                        | Associated boost has status=`FAILED`                           |
| 9 | Dismissed                     | status=`dismissed`                                             |

**First-class queue items besides Actions:**
- *Custom workflows* (orphan boosts with no `actionId`) interleaved into the list (`page.tsx:699`).
- *Pitch outlets* — recommended platforms, each selectable with its own pane (`page.tsx:700`).

**Search** is a substring match on `title + reason` for actions, `name + goal` for boosts, `label + domain + competitors` for pitches (`page.tsx:365–425`).

**Keyboard nav** (`page.tsx:494–516`):
- `j` / `↓` next action, `k` / `↑` previous (sort follows queue stage order, then priority/createdAt)
- `Esc` close drawer/selection
- `?` toggle help overlay

### 2.4 ActionDetailPane

`frontend/components/beacon/action-detail-pane.tsx:170–650+`.

**Header**
- Priority pill (HIGH=red, MEDIUM=amber, LOW=blue) with hover tooltip showing the priority math when `metadata.priorityExplanation` is set (`action-detail-pane.tsx:314–322`).
- Impact-area icon (eye / doc / chart / target).
- Top-threat badge (competitor name from `metadata.topThreat`).
- Channel badge from `CHANNEL_BADGE[channelKey]`.
- Status dropdown (`open` / `completed` / `dismissed`) — change fires `updateStatus` mutation.

**Evidence section** is fetched lazily depending on `impactArea` (`action-detail-pane.tsx:244–250`):
- `sentiment` → `GET /beacon/company/:id/trust`
- `comparison` → `GET /beacon/company/:id/comparison`
- everything else → `GET /beacon/company/:id/discovery`

Cached for 60s. Affected prompts come from `metadata.affectedPrompts` if Engine B populated it; otherwise extracted from this evidence response (`action-detail-pane.tsx:255–281`).

**Proposals section** (`PerActionProposals`)
Renders each proposal as a selectable card showing playType, target channel, estimated impact, rationale. If proposals are empty AND no plan is running → "Generate action proposals" button, scoped to this action (`focusActionId` saves orchestrator tokens).

**Live tool-call indicator**
While a plan is running, polls `GET /content-lab/orchestrator-run/:id` every **1.5s** and shows the latest tool name (e.g. `"running · get_evidence"`) beside the spinner (`action-detail-pane.tsx:217–236`).

**Auto-trigger on open**
First time the user opens an action whose company has no in-flight plan, the page auto-fires a plan-mode orchestrator run with a **6h cooldown guard** so re-clicking the same action doesn't burn tokens.

### 2.5 Play renderers

`frontend/components/beacon/play-cards.tsx` exports `PlayDraftRenderer`, which routes a `ContentDraft` to a per-platform UI based on `playType`:

| Renderer            | playTypes it handles                                    |
|---------------------|----------------------------------------------------------|
| `BlogViewer`        | `blog_post`, `pillar_page`, `faq_page`, `trust_page`, `comparison_article`, `case_study`, `product_page`, `lead_magnet` |
| `RedditCard`        | `reddit_response`                                        |
| `LinkedInCard`      | `linkedin_post`                                          |
| `TwitterCard`       | `twitter_thread`                                         |
| `QuoraCard`         | `quora_answer`                                           |
| `ForumCard`         | `forum_response`, `community_post`                       |
| `EmailDraftCard`    | `email_pitch`, `b2b_outreach`                            |
| `DirectoryEntryCard`| `directory_submit`, `marketplace_claim`, `wiki_edit`, `award_submit`, `review_solicit`, `listicle_pitch` |

Each renderer also displays the per-domain "where to deploy" buttons (TripAdvisor, G2, Reddit, Quora, etc.) replacing the generic playbook copy that used to render here (commit `88f2a79`).

### 2.6 ActionEvent funnel

`frontend/lib/action-events.ts:1–52` exports `recordActionEvent({ companyId, actionId, kind, metadata })`. Fire-and-forget, never throws (POSTs to `/beacon/company/:id/action-event`).

Kinds emitted from the page today:

| Kind                | Where                                                    |
|---------------------|----------------------------------------------------------|
| `opened`            | `setSelectedKey` deduped per page-load (`page.tsx:60–63`)|
| `draft_viewed`      | When the content modal opens with an attributable `actionId` (`page.tsx:90–101`) |
| `draft_copied`      | Copy-to-clipboard inside `PlayDraftRenderer`             |
| `action_completed`  | `updateStatus` mutation success path (`page.tsx:276–279`)|
| `action_dismissed`  | same mutation                                            |
| `plan_started` / `plan_completed` | fired backend-side from the orchestrator (§5)|

Backend lookup + insert lives in `backend/src/services/action-event.service.ts:61–98`. Validates `kind` against `KNOWN_KINDS` (`action-event.service.ts:23–33`), looks up `companyId` if absent, returns row id or `null` on failure.

### 2.7 MissionControl + Run Autopilot modal

`frontend/components/beacon/mission-control.tsx`. Displays the current orchestrator-run state (queued / planning / running / idle / failed) with a pulsing dot and "last run X ago" stamp. Click "Run now" opens the pre-flight modal where the user can override goal/context, then POST `/content-lab/company/:id/orchestrate`. Stuck-run detection fires after 5 minutes (modal offers cancel via `POST /content-lab/orchestrator-run/:id/cancel`).

### 2.8 KPI strip data flow

```
allActions ──▶ derive: highPriorityCount, recentlyResolved
              visibilityActions  ─▶ estimatedUplift = +min(N×3, 25)%
boosts     ──▶ derive: runningBoostsCount
              ▼
       SummaryStrip (4 cards, hover for definition keyed off lib/metric-definitions.ts)
```

Numerical formulas live in §9.

### 2.9 React Query keys (the page's working set)

| Key                                                         | TTL / refetch            | Purpose                          |
|-------------------------------------------------------------|--------------------------|----------------------------------|
| `['beacon', cId, 'actions', 'all', timeRange, model]`       | 30s                      | All actions (queue + KPIs)       |
| `['boosts', cId]`                                           | 30s interval             | Boosts (orphan + scoped)         |
| `['recommended-platforms', cId]`                            | 5m                       | Pitch outlets                    |
| `['orchestrator-runs', cId]`                                | dynamic (3s if running)  | Plan/execute history             |
| `['content-drafts', cId]`                                   | 30s                      | Persisted drafts                 |
| `['beacon', cId, 'action-evidence', actionId, impactArea]`  | 60s                      | Evidence for detail pane         |
| `['orchestrator-run', runId, 'live']`                       | 1.5s while running       | Tool-call indicator              |
| `['orchestrator-run', executeRunId]`                        | 4s while executing       | Execute progress                 |

No optimistic updates anywhere — every mutation invalidates and refetches.

---

## 3. How actions are generated (Engine B)

### 3.1 Entry & batch-load

**File:** `backend/src/services/action-intelligence.service.ts:298+` (`generateIntelligentActions(companyId)`).

Six parallel Prisma queries pre-load the dataset (`action-intelligence.service.ts:298–328`):

```ts
Promise.all([
  prisma.company.findUnique(...),                // name, websiteUrl
  prisma.competitor.findMany(...),               // tracked competitors
  prisma.simulation.findMany({ take: 500, ... }),// 500 newest sims w/ prompt join
  prisma.citationSource.findMany(...),
  prisma.citation.findMany({                     // joined to simulation
    select: { ..., simulation: { select: { mentionsCompany, presenceQuality, ... } } }
  }),
  prisma.domainOverride.findMany(...),
])
```

After load it pre-classifies every cited source by outreach channel (`action-intelligence.service.ts:336–344`) using the deterministic `classifyDomain` helper from `lib/channel-classifier.ts`, so downstream lanes and the Sources page agree on taxonomy.

### 3.2 Lanes (each independent, all run inside one call)

| # | Lane                               | Function                              | Output evidenceType    |
|---|------------------------------------|---------------------------------------|------------------------|
| 1 | Prompt-gap (discovery + comparison)| `analyzePromptGaps`                   | `prompt_gap`           |
| 2 | Sentiment / hedging objections     | `mineObjections` / sentiment lane     | `objection`            |
| 3 | Citation DNA (per-source)          | `analyzeCitationDNA`                  | `citation_dna`         |
| 4 | Channel lane / hub / momentum      | per-channel sub-functions             | `citation_dna`         |
| 5 | Cross-model divergence             | `detectModelDivergence`               | `model_divergence`     |
| 6 | Long-tail surface scout (Phase 5.1)| `mineLongTailSurfaces` → `scoutSource`| `surface_scout`        |
| 7 | Hub / topic gaps                   | `analyzeHubGaps`, `analyzeTopicGaps`  | `hub_gap`              |

Each lane appends `ActionSeed[]` to a single accumulator, then the engine sorts → dedupes → caps → returns.

### 3.3 Shared scoring inputs

All defined near the top of the file:

```ts
const CATEGORY_WEIGHTS = { discovery: 1.0, comparison: 1.2, trust_validation: 1.5, other: 1.0 }   // :156
const COMPETITOR_PRESENCE_SCORE = { PRIMARY: 1.0, SECONDARY: 0.7, MENTION: 0.35, ABSENT: 0 }       // :170
const UNREACHABLE_REASONS = new Set(['http_403', 'http_404', 'http_5xx', 'fetch_failed'])          // :117
```

Trust-tier weights for Citation DNA: `TIER_1 → 3, TIER_2 → 2, TIER_3 → 1`.

`parseCompetitorMentions(raw)` (`:200–221`) parses the `mentionsCompetitors` JSON column into `{ name, mentioned, position, presenceQuality }[]`.

### 3.4 Lane 1 — `analyzePromptGaps`

**Inputs:** sims filtered to `category ∈ {discovery, comparison}` (`action-intelligence.service.ts:410–413`).

**Per-prompt aggregation** (`:444–502`):
- `lossRuns`: discovery → company missing AND ≥1 competitor mentioned. comparison → `analysisData.comparisonOutcome === 'loss'` OR same as discovery rule.
- For every mentioned competitor: `count`, `primaryWins` (`presenceQuality === 'PRIMARY'` or `position === 1`), `bestPosition`.
- `companyMentionRate` = sims with `mentionsCompany / total`.
- `primaryLossRate` = `topCompetitor.primaryWins / total`.

**Keep filter** (`:507–510`):
- discovery: keep only if `companyMentionRate === 0` (we're absent in ALL runs).
- comparison: keep if `lossRuns > 0` and (`lossRuns/total ≥ 0.5` OR `primaryLossRate ≥ 0.5`).

**Per-prompt weightedRisk** (`:514–517`):
```
weightedRisk = lossRuns * categoryWeight
             + topCompetitor.primaryWins * 1.6
             + max(0, competitors.length - 1) * 0.35
```
Magic numbers: **1.6** primary-win penalty, **0.35** crowded-answer penalty. Their rationale is documented in §9.5; they are eyeballed, not regression-fit.

**Topic-cluster aggregation** (`:537–569` discovery, `:608–637` comparison):
```
revenueScore = round(Σ weightedRisk for the cluster) * 10           // integer
primaryLossRate = Σ primaryWins / Σ totalRuns
```

**Priority** (`:573` discovery, `:639` comparison):
- Discovery HIGH if `revenueScore ≥ 45` OR `primaryLossRate ≥ 0.6`.
- Comparison HIGH if `revenueScore ≥ 40` OR `primaryLossRate ≥ 0.55`.
- Else MEDIUM.

**Seed shape** for prompt-gap actions:
```jsonc
{
  "priority": "HIGH" | "MEDIUM",
  "title": "Revenue risk: <topic> <intent> demand goes to <competitor>",
  "reason": "<sample prompt + dashboard metric>",
  "impactArea": "visibility",
  "evidenceType": "prompt_gap",
  "metadata": {
    "intentSegment": "discovery" | "comparison",
    "revenueScore": 50,
    "primaryLossRate": 0.62,
    "priorityExplanation": { "score": 50, "threshold": 45, "components": { ... }, "why": "..." },
    "affectedPrompts": [{ "id", "text", "detail", "lostTo" }],
    "topThreat": { "name": "...", "promptCount": 3, "promptIds": [...] },
    "playType": "blog_post" | "comparison_article",
    "contentBrief": { "suggestedTitle", "targetKeywords", "outline", "estimatedImpact" }
  }
}
```

### 3.5 Lane 2 — sentiment / objection mining

**Inputs:** sims where `mentionsCompany=true` AND `responseText` exists (`:996+`).

For each response, `extractSentences()` (`:240–243`) splits on `.!?` boundaries and filters out fragments. Each sentence is tested against `HEDGING_PATTERNS` (`:121–131`): "however", "some users report", "one concern", "drawback", "can be expensive", "mixed reviews", etc. Hedged sentences are then classified into a theme via `THEME_PATTERNS` (`:134–143`):

| Theme       | Trigger words (regex)                                             |
|-------------|-------------------------------------------------------------------|
| pricing     | price, cost, expensive, budget, premium, fees                     |
| quality     | quality, mediocre, inconsistent, subpar, outdated                 |
| service     | service, staff, support, slow, rude                               |
| features    | feature, missing, lack, limited, basic                            |
| experience  | UI, UX, confus, clunky, complicated                               |
| reliability | reliab, bug, crash, downtime, error, unstable                     |
| location    | location, area, access, parking, remote                           |
| competition | competitor, alternative, versus, better option                    |

**Scoring**:
```
frequency = uniqueSimsWithTheme / totalMentionedSims * 100
```
Skip if `frequency < 10` AND `uniqueSims < 3`.

**Priority**:
- `frequency ≥ 50%` → HIGH
- `frequency ≥ 25%` → MEDIUM
- else → LOW

### 3.6 Lane 3+4 — Citation DNA

**Inputs:** every cited source, joined to its simulation outcome.

For each `CitationSource`: `{ domain, url, trustTier, citationCount, companyMentioned, companyAbsent, companyPrimary, totalSims, events[], competitorCounts }`.

**Hub score** (top-of-funnel — competitor-only sources):
```
hubScore = trustWeight[trustTier] * citationCount * max(1, competitorHubCount)
```
HIGH if `competitorHubCount ≥ 4` OR `consensusScore ≥ 0.8`.

**Momentum detection** (rank regression):
- Split timeline events into recent half (newest ⌈N/2⌉) and previous half.
- Need ≥4 events with ≥2 in each half.
- Flag if `previousMentionRate ≥ 0.5` AND `recentMentionRate ≤ 0.25` AND `recentPrimaryAbsence ≥ 2`.
- Score: `(prev - recent) * trustWeight * citationCount`.

**Per-source `pickPriority`** (used across channel lanes, line ~993):
- HIGH if `trustTier = TIER_1` OR (`TIER_2` AND `citationCount ≥ 3`) OR (`citationCount ≥ 5`).
- MEDIUM if `TIER_3` AND `citationCount ≥ 2`.
- LOW otherwise.

**Channel lanes** (one action per channel where `gapDomains > 0` and (`sources.length ≥ 2` OR `totalCitations ≥ 8`)):

| ChannelKey         | playType             | Action title pattern                                  |
|--------------------|----------------------|-------------------------------------------------------|
| EDITORIAL          | `email_pitch`         | "Pitch <outlet> on …"                                 |
| LISTICLE           | `listicle_pitch`      | "Get added to <listicle>"                             |
| COMMUNITY_FORUM    | `reddit_response` / `quora_answer` / `forum_response` / `community_post` | dynamic |
| MARKETPLACE        | `marketplace_claim`   | "Claim listing on <market>"                           |
| REVIEW_PLATFORM    | `review_solicit`      | "Solicit reviews on <platform>"                       |
| PARTNER_AGENCY     | `b2b_outreach`        | "B2B outreach to …"                                   |
| DIRECTORY          | `directory_submit`    | "Submit to <directory>"                               |
| WIKI               | `wiki_edit`           | "Edit <wiki>"                                         |
| AWARDS             | `award_submit`        | "Apply: <award>"                                      |
| OWN_DOMAIN         | `on_domain_content`   | "Publish: …"                                          |
| COMPETITOR         | `comparison_article`  | "Comparison page: us vs <competitor>"                 |

`UNCATEGORIZED` channels are routed to Lane 6 (surface scout).

### 3.7 Lane 5 — model divergence

For each discovery prompt, build a per-model best-mention map. Two patterns are surfaced:

- **Consensus gap**: model A absent while ≥2 peers mention. HIGH if `gaps.length ≥ 4` OR `avgConsensusScore ≥ 0.8`.
- **Broad visibility gap**: minority of models miss a query the majority answers. HIGH if `gaps.length ≥ 4`.

Seed metadata carries `models`, `consensusScore`, and per-prompt `lostModels`.

### 3.8 Lane 6 — long-tail surface scout

`mineLongTailSurfaces` (`action-intelligence.service.ts:850+`) shortlists **UNCATEGORIZED** sources where the company is absent (≥30% absence rate OR ≥2 absolute absences), scores them with `(trustWeight × citationCount) + absentCount`, and caps at `SURFACE_SCOUT_PER_COMPANY_CAP = 15` (`surface-scout.ts:39`). Scout uses a *steeper* trustWeight than the rest of Engine B: `TIER_1=4, TIER_2=2, TIER_3=1` (`:886`), to surface high-authority long-tail domains harder. **No floor on `citationCount`** any more — even a 1-cite long-tail domain qualifies (relaxed 2026-05-02 because tenants like Payit had 28/28 long-tail domains at 1–2 cites).

For each candidate it calls `scoutSource()` from `backend/src/agents/sub-agents/surface-scout.ts`:

1. `fetchPageText()` — axios GET, 8s timeout, plain-text up to 12,000 chars.
2. `summarisePage()` — Haiku (`claude-haiku-4-5-20251001`) summary in 70–90 words.
3. `generateCandidates()` — Sonnet returns exactly 3 ranked actions (each 5–60 minutes) as JSON.

Result becomes a `surface_scout` action with `metadata.scoutCandidates: { action, estimatedMinutes, why }[]`.

### 3.9 Sort, dedupe, cap

After all lanes (`action-intelligence.service.ts:374–397`):

1. **Sort**: priority HIGH→MED→LOW, then `getActionStrength()` desc, then title.
2. **Dedup layer 1** (`domain::playType`): same domain + play type only emits once.
3. **Dedup layer 2** (title hash): catches divergence-style actions without a domain.
4. **Cap**: top **30** intelligent actions returned to the caller (`:525`, bumped 15→30 on 2026-05-02 to surface a larger long-tail set; downstream critic + 3-layer regenerator dedup typically halves volume back to ~15 in the queue).

`getActionStrength` (`action-intelligence.service.ts:280–288`) is the strength score used in the dedupe sort:
```
strength = (riskScore ?? revenueScore ?? 0)
         + citationPower * 3
         + competitorHubCount * 8
         + round(consensusScore * 100)
         + round((prevMentionRate - recentMentionRate) * 100)
         + round((modelNegativeRate - peerNegativeRate) * 100)
         + round(channelOpportunityScore)
         + targetDomains.length * 2
```

### 3.10 Channel → playType routing

The `playTypeForChannel(channel, domain)` switch (`action-intelligence.service.ts:80–111`) is the single source of truth for "what content do we make for this channel?" — see the channel table in §3.6.

---

### 3.11 Action catalog — every emitter, every threshold (scout vs non-scout)

This is the exhaustive map: **every** `actions.push({...})` site in `action-intelligence.service.ts` plus the surface-scout emitter, the gate that has to pass before the push fires, the priority rule, and what "score" the priority is computed from. If you ever ask "why did this action show up as HIGH?", this section is the answer.

> **Convention.** Engine B emits **20 distinct action shapes** across 6 lanes. They split into two families that have different prompt-vs-source provenance:
> - **Non-scout actions (19)** — derived deterministically from in-DB simulation/citation data. No LLM call to generate the action itself.
> - **Scout actions (1, but 1-per-domain)** — derived by an LLM (Haiku summary + Sonnet candidates) reading the cited page. The action carries 3 user-pickable sub-candidates inside `metadata.scoutCandidates`.

#### 3.11.1 Shared helpers used below

```ts
// Citation-DNA file-local helpers (all defined inside analyzeCitationDNA, :1210–1334)
trustWeight        = { TIER_1: 3, TIER_2: 2, TIER_3: 1 }       // :1211 (DNA lane)
trustWeight (Scout)= { TIER_1: 4, TIER_2: 2, TIER_3: 1 }       // :886  (scout lane only — heavier TIER_1)
tierOrder          = { TIER_1: 0, TIER_2: 1, TIER_3: 2 }       // :1210 (used as final tiebreaker in sort)

lossRate(s)        = s.totalSims > 0 ? s.companyAbsent / s.totalSims : 0           // :1212
lossRatePct(s)     = round(lossRate(s) * 100)                                       // :1298

hubScore(s)        = trustWeight[s.trustTier] * s.citationCount * max(1, competitorHubCount)   // :1237

momentum.dropScore = (previousMentionRate − recentMentionRate) * trustWeight × citationCount   // :1254
momentum gate      = previousMentionRate ≥ 0.5 AND recentMentionRate ≤ 0.25
                     AND recentPrimaryAbsence ≥ 2                                   // :1257

competitorSources order: by [hubCount×2 + (momentum?3:0) + trustWeight×2 + citationCount + lossRate]
                          tiebreak tier ASC, citationCount DESC                     // :1278

sourceOpportunityScore(s) =
   trustWeight[s.trustTier] * max(1, s.citationCount) * max(0.35, lossRate(s))
   + (competitorHubCount * 3)
   + (momentum ? 6 : 0)                                                              // :1331

pickPriority(s):                                                                    // :1299
  TIER_1                          → HIGH
  TIER_2 AND citationCount ≥ 3    → HIGH
  citationCount ≥ 5               → HIGH
  TIER_3 AND citationCount < 2    → LOW
  else                            → MEDIUM
```

**Two competitor-source pre-filters (apply before any push in this lane):**
- `companyAbsent > companyMentioned` (we lose more than we win on this source) AND `totalSims ≥ 2` (`:1274`).
- `channel ≠ OWN_DOMAIN` (don't pitch yourself), AND (channel = COMPETITOR OR `mentionReason ∉ UNREACHABLE_REASONS`) (`:1275–1277`).

`UNREACHABLE_REASONS = { http_403, http_404, http_5xx, fetch_failed }` (`:117`).

---

#### 3.11.2 Lane 1 — Prompt gap (non-scout)

| # | Emitter (line)                                                | Gate to fire                                                                                                       | Priority rule                                              | What's in the title                                       |
|---|---------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------|------------------------------------------------------------|
| 1 | **Discovery prompt-gap cluster** (`:709`)                     | A topic cluster exists with `companyMentionRate === 0` for every prompt in it. Picks the highest `revenueScore` topic. | `revenueScore ≥ 45` OR `primaryLossRate ≥ 0.6` → HIGH; else MEDIUM | `Revenue risk: <topic> discovery demand goes to <competitor>` |
| 2 | **Comparison prompt-gap cluster** (`:787`)                    | Comparison gaps exist; `lossRuns/total ≥ 0.5` OR `primaryLossRate ≥ 0.5` per prompt. Picks highest-`revenueScore` competitor group. | `revenueScore ≥ 40` OR `primaryLossRate ≥ 0.55` → HIGH; else MEDIUM | `Revenue protection: <competitor> shapes buyer comparisons` |

Both share the per-prompt formula:
```
weightedRisk = lossRuns × CATEGORY_WEIGHTS[category]              // discovery 1.0, comparison 1.2, trust 1.5
             + primaryWins × 1.6
             + max(0, competitors.length − 1) × 0.35
revenueScore (cluster) = round(Σ weightedRisk × 10)               // integer
primaryLossRate (cluster) = Σ primaryWins / Σ totalRuns
```

`priorityExplanation` is populated for both (`:723`, `:802`) so the UI tooltip can spell out which threshold tripped.

---

#### 3.11.3 Lane 2 — Sentiment / objection (non-scout)

| # | Emitter (line)                          | Gate to fire                                                                          | Priority rule                                | What's in the title                                       |
|---|------------------------------------------|----------------------------------------------------------------------------------------|------------------------------------------------|------------------------------------------------------------|
| 3 | **Objection theme cluster** (`:1079`)   | At least one HEDGING_PATTERN match across sims; theme cluster passes `frequency ≥ 10%` AND `uniqueSims ≥ 3`. | `frequency ≥ 50` → HIGH, `≥ 25` → MEDIUM, else LOW | `Strengthen <theme> credibility for <company>`            |

Where `frequency = uniqueSimsWithTheme / totalMentionedSims × 100`. Themes come from `THEME_PATTERNS`: pricing, quality, service, features, experience, reliability, location, competition (`:209–218`).

---

#### 3.11.4 Lane 3 — Citation DNA + channel lanes (non-scout)

All operate on the `competitorSources` set defined above. **Each per-channel emitter caps at 1–4 sources** so the queue isn't flooded by one channel.

| # | Emitter (line)                                                          | Gate to fire                                                                                                                                                     | Priority rule                                                                | Channel slice |
|---|--------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------|---------------|
| 4 | **Channel strategy bucket** (`:1376`)                                   | Channel ∉ {OWN_DOMAIN, UNCATEGORIZED}. Bucket has `gapDomains > 0` AND (`sources ≥ 2` OR `totalCitations ≥ 8`). Top 2 buckets by `Σ sourceOpportunityScore`.        | `bucket.opportunityScore ≥ 18` OR `leadSource.trustTier === 'TIER_1'` → HIGH | top 2 buckets, top 4 sources each |
| 5 | **Competitor citation hub** (`:1408`)                                    | Source is in `competitorSources` AND `competitorHubCount ≥ 2`; pick top 3 by score.                                                                                | Any source TIER_1 OR `coveredCompetitors ≥ 3` → HIGH; else MEDIUM           | up to 3 hubs in one action |
| 6 | **Momentum loss** (`:1451`)                                              | Source has a `momentumBySourceId` entry (i.e. `prevRate ≥ 0.5 AND recentRate ≤ 0.25 AND recentPrimaryAbsence ≥ 2`); top 2.                                          | `trustTier === 'TIER_1'` OR `previousMentionRate ≥ 0.75` → HIGH; else MEDIUM | one per source, max 2 |
| 7 | **EDITORIAL pitch** (`:1480`)                                            | Source channel = EDITORIAL.                                                                                                                                       | `pickPriority(s)`                                                            | top 2 |
| 8 | **LISTICLE inclusion** (`:1499`)                                         | Source channel = LISTICLE.                                                                                                                                        | `pickPriority(s)`                                                            | top 2 |
| 9 | **COMMUNITY/SOCIAL post** (`:1521`)                                      | Source channel ∈ {COMMUNITY_FORUM, SOCIAL_PLATFORM}.                                                                                                              | `pickPriority(s)`                                                            | top 2 combined |
| 10| **DIRECTORY claim** (`:1541`)                                            | Source channel = DIRECTORY.                                                                                                                                       | `pickPriority(s)`                                                            | top 1 |
| 11| **REVIEW_PLATFORM solicit** (`:1541`, in submissionChannels loop)        | Source channel = REVIEW_PLATFORM.                                                                                                                                 | `pickPriority(s)`                                                            | top 1 |
| 12| **WIKI edit** (`:1541`)                                                  | Source channel = WIKI.                                                                                                                                            | `pickPriority(s)`                                                            | top 1 |
| 13| **AWARDS submission** (`:1541`)                                          | Source channel = AWARDS.                                                                                                                                          | `pickPriority(s)`                                                            | top 1 |
| 14| **PARTNER_AGENCY B2B outreach** (`:1556`)                               | Source channel = PARTNER_AGENCY.                                                                                                                                  | `pickPriority(s)`                                                            | top 2 |
| 15| **COMPETITOR — defensive comparison** (`:1570`)                         | Source channel = COMPETITOR (set deterministically by classifier).                                                                                                 | `trustTier === 'TIER_1'` OR `citationCount ≥ 5` → HIGH; else MEDIUM         | top 2 |
| 16| **Catch-all "get featured"** (`:1607`)                                   | Source not yet consumed by a channel-specific emitter, passes the LLM `filterCompetitorDomains()` safe filter.                                                     | `pickPriority(s)`                                                            | top 3 of remainder |
| 17| **LLM-blocked competitor — defensive comparison** (`:1626`)             | Source was rejected by `filterCompetitorDomains()` as competitor-like.                                                                                            | `trustTier === 'TIER_1'` OR `citationCount ≥ 5` → HIGH; else MEDIUM         | top 3 of blocked |
| 18| **Winning sources pattern** (`:1670`)                                    | `≥ 1` source where `companyPrimary > 0` AND `losingSources > 3` total in tenant.                                                                                  | constant **MEDIUM**                                                          | up to 5 winning domains in one action |

**Why 18 / TIER_1 / hub thresholds?**
- `opportunityScore ≥ 18` for a bucket means roughly: (TIER_2 source × 3 cites × 0.6 lossrate ≈ 3.6) + (1 hub × 3) + (1 momentum × 6) = ~12.6 baseline, so a bucket has to be measurably better than just one decent source. TIER_1 always promotes a bucket regardless of score because a single TIER_1 cite outweighs five TIER_3 cites.
- `coveredCompetitors ≥ 3` for hubs catches sources where ≥3 of *your tracked rivals* show up — a definitional citation hub.
- `previousMentionRate ≥ 0.75` for momentum is a "we used to be the obvious answer" floor; combined with the 0.5/0.25 gate it ensures a real ranking regression rather than noise.

`buildSourceMetadata(source, overrides)` (`:1314`) populates `citationDomain / citationUrl / citationPower / channelKey / channelLabel / channelAction / playType / targetDomains` for emitters #7–17. `channelOpportunityScore = round(opportunityScore × 10)` is stamped onto emitter #4 only.

---

#### 3.11.5 Lane 5 — Cross-model divergence (non-scout)

| # | Emitter (line)                                            | Gate to fire                                                                                                                       | Priority rule                                            | What's in the title                                              |
|---|------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------|--------------------------------------------------------------------|
| 19a | **Consensus visibility gap** (`:1798`)                  | Per discovery prompt: `mentionedEntries ≥ 2` AND `absentEntries.length === 1`. Aggregated per absent model with `gaps ≥ 2`.         | `gaps ≥ 4` OR `consensusScore ≥ 0.8` → HIGH; else MEDIUM | `Consensus gap on <model>: peers mention you, it doesn't`         |
| 19b | **Broad visibility gap** (`:1822`, **fallback only**)   | Fires only if no consensus actions emitted. Per absent model, `gaps ≥ 2`.                                                            | `gaps ≥ 4` → HIGH; else MEDIUM                          | `Visibility gap on <model>: it misses queries peers can answer`   |
| 20a | **Sentiment outlier — lone-negative** (`:1942`)         | trust_validation/comparison sims with sentiment != null AND `mentionsCompany`. Per prompt `≥ 2` mentioned. Lone-negative model with `gaps ≥ 2`. | `gaps ≥ 4` OR `consensusScore ≥ 0.75` → HIGH; else MEDIUM | `Brand health gap on <model>: it turns positive prompts negative`  |
| 20b | **Hostility watch** (`:1987`)                            | Pre-filter `total ≥ 4` AND `ownNegativeRate ≥ 0.35` AND `ownNegativeRate ≥ peerNegativeRate + 0.2`. Picks worst by `(own − peer)`.    | `ownNegativeRate ≥ 0.5` → HIGH; else MEDIUM             | `Model bias watch: <model> is materially harsher than peer models` |
| 20c | **Generic sentiment split** (`:2008`, **fallback only**)| Fires only if hostility didn't, AND `generalSentimentSplits ≥ 2`.                                                                    | constant **MEDIUM**                                      | `Sentiment split: <models> negative while peers are positive`     |

Note the **"only if no consensus" / "only if hostility didn't"** fallback gates — at most one visibility-gap action and at most one sentiment-divergence action per lane per run.

---

#### 3.11.6 Lane 6 — Surface scout (the only LLM-driven emitter)

This is the **only** action whose generation requires a live external HTTP fetch + two LLM calls per candidate domain.

| # | Emitter (file)                                                        | Gate to fire                                                                                                                                                    | Priority rule                                                                                | Title pattern                                                          |
|---|-------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------|------------------------------------------------------------------------|
| 21 | **`buildSurfaceScoutSeed`** (`backend/src/agents/sub-agents/surface-scout.ts:269`) | Source channel = UNCATEGORIZED AND `isWeakMention(s)` (≥30% absent OR ≥2 absent). Top 15 by `(scoutTrustWeight × citationCount) + absentCount`. Each must produce a non-null `scoutSource()` result. | `TIER_1` → HIGH; `TIER_2 AND citationCount ≥ 3` → HIGH; `citationCount ≥ 5` → HIGH; else MEDIUM | `Investigate <domain>: <first candidate action, sliced to 60 chars>` |

Per-candidate generation (`scoutSource`, `:241`):

1. **`fetchPageText(url)`** (`:94–131`) — axios GET, 8s timeout, plain text, capped at 12k chars. Returns `null` on non-HTML / timeouts → emitter skips.
2. **`summarisePage(input, pageText)`** (`:136–161`) — Haiku `claude-haiku-4-5-20251001`, 70–90 word summary of what the page is, who it's for, why it cites the brand.
3. **`generateCandidates(input, summary)`** (`:165–231`) — Sonnet `claude-sonnet-4-6`, returns **exactly 3** ranked candidates as JSON, each `{ action, estimatedMinutes (5–60), why }`.

If any of the three fails, the emitter logs and skips the source rather than producing a partial action. The detail pane reads `metadata.scoutCandidates` to render a 3-radio picker.

**Scoring math (gate-side, before Haiku/Sonnet runs):**
```
score(s) = trustWeight[s.trustTier] * s.citationCount + absentByDomain[s.domain]
            // trustWeight steeper here: TIER_1=4, TIER_2=2, TIER_3=1
```
The score is *only* used to pick the top-15 candidates — the priority of the resulting action is set by `buildSurfaceScoutSeed` separately and ignores this score.

**Anchoring side-effect.** Surface-scout actions are special at proposal-time: any proposal targeting a `surface_scout` action must point at the scoutDomain itself, the client's own domain, or one of `ANCHOR_COMMUNITY_DOMAINS = {reddit.com, quora.com, linkedin.com, x.com, twitter.com, medium.com}` — see §6.2. This is enforced in `proposal-validator.ts:319–357` and rejects "unanchored" proposals that wandered off-target.

---

#### 3.11.7 Cheat sheet — all 20 thresholds at a glance

```
Lane 1 — discovery cluster        HIGH:  revenueScore ≥ 45  OR  primaryLossRate ≥ 0.6
         comparison cluster       HIGH:  revenueScore ≥ 40  OR  primaryLossRate ≥ 0.55
Lane 2 — objection                 HIGH:  frequency% ≥ 50    MEDIUM: ≥ 25      LOW: < 25 (but kept ≥ 10 with ≥3 sims)
Lane 3 — channel bucket            HIGH:  opportunityScore ≥ 18  OR  leadTier == TIER_1
         hub                       HIGH:  any source TIER_1     OR  coveredCompetitors ≥ 3
         momentum                  HIGH:  TIER_1 OR previousMentionRate ≥ 0.75
         pickPriority (most)       HIGH:  TIER_1 || (TIER_2 && cites ≥ 3) || cites ≥ 5
                                   LOW:   TIER_3 && cites < 2
                                   else MEDIUM
         COMPETITOR (deterministic)HIGH:  TIER_1 OR cites ≥ 5
         COMPETITOR (LLM-blocked)  HIGH:  TIER_1 OR cites ≥ 5
         winning sources           CONSTANT MEDIUM
Lane 5 — consensus gap             HIGH:  gaps ≥ 4 OR consensusScore ≥ 0.8
         broad gap (fallback)      HIGH:  gaps ≥ 4
         sentiment outlier         HIGH:  gaps ≥ 4 OR consensusScore ≥ 0.75
         hostility watch           HIGH:  ownNegativeRate ≥ 0.5
                                   (gate: total ≥ 4 AND ownRate ≥ 0.35 AND ownRate ≥ peerRate + 0.2)
         sentiment split (fallback)CONSTANT MEDIUM
Lane 6 — surface scout             HIGH:  TIER_1 || (TIER_2 && cites ≥ 3) || cites ≥ 5
                                   else MEDIUM (no LOW)
                                   gate: UNCATEGORIZED + (≥30% absent OR ≥2 absent)
                                   cap : 15 candidates, 3 sub-actions each
```

#### 3.11.8 Strength-based dedupe ordering

When two emitters produce overlapping actions, `getActionStrength` (`:385`) decides who wins the dedupe:
```
strength = (riskScore ?? revenueScore ?? 0)             // Lane 1
         + citationPower * 3                            // Lanes 3, 6
         + competitorHubCount * 8                       // Lane 3 hub
         + round(consensusScore * 100)                  // Lane 5
         + round((prevMentionRate − recentMentionRate) * 100)   // Lane 3 momentum
         + round((modelNegativeRate − peerNegativeRate) * 100)  // Lane 5 hostility
         + round(channelOpportunityScore)               // Lane 3 channel bucket
         + targetDomains.length * 2
```
This is why a momentum action on a TIER_1 cite-hub will outscore (and survive over) a generic per-channel pitch action targeting the same domain.

---

## 4. Persistence: from `ActionSeed` to `Action` row

### 4.1 The Action model

`backend/prisma/schema.prisma`:
```prisma
model Action {
  id           String  @id @default(uuid())
  companyId    String
  priority     String  @default("MEDIUM")     // HIGH | MEDIUM | LOW
  title        String
  reason       String  @db.Text
  impactArea   String  @default("general")    // visibility | citations | sentiment | general
  evidenceType String  @default("manual")     // prompt_gap | objection | citation_dna | model_divergence | surface_scout | hub_gap | manual
  metadata     Json?
  status       String  @default("open")       // open | completed | dismissed | in_progress
  completedAt  DateTime?
  createdAt    DateTime @default(now())
  updatedAt    DateTime @updatedAt
  company  Company         @relation(fields: [companyId], references: [id], onDelete: Cascade)
  boosts   Boost[]
  events   ActionEvent[]
  outcomes ActionOutcome[]
  @@index([companyId])
  @@index([status])
  @@index([priority])
}
```

`metadata` is the catch-all JSON column — most analytics live in there.

### 4.2 `regenerateActionsForCompany`

Triggered by **`POST /beacon/company/:id/actions/generate`** (the "Scan for actions" CTA) and by the staleness check inside the actions GET handler.

Lives in `backend/src/routes/beacon.routes.ts:2034+`. Pseudocode:

```
existing = prisma.action.findMany({ where: { companyId, status: 'open' } })
intelligent = generateIntelligentActions(companyId)        // Engine B
promptSet = generatePerPromptActions(companyId)            // Engine A
candidates = [...promptSet, ...intelligent]

dedupe in three layers:
  1. dedupeHash OR title vs existing
  2. target-domain (no two open actions targeting the same domain)
  3. normalized contentBrief.suggestedTitle (Phase 4.5.3)

cap citation_dna to 8                                       // Phase 2.2 audit fix
prisma.action.createMany(survivors)
prisma.action.updateMany(touched_existing) // SQL JSONB merge
```

`dedupeHash` (`action-intelligence.service.ts:275–298`) is `SHA256(impactArea | evidenceType | playType | sorted(domains) | sorted(promptIds)).slice(0, 32)` — deterministic across runs so the same evidence never spawns two rows.

### 4.3 Touch-only update path

The "touch existing" pass uses raw SQL to JSONB-merge the new `metadata` and bump `updatedAt` + `generatorVersion` without going through `prisma.action.update` per row (which exhausted the connection pool on big tenants).

---

## 5. The orchestrator and its sub-agents

### 5.1 Trigger

User opens an Action → page POSTs `/content-lab/company/:id/orchestrate` with `{ mode: 'plan', focusActionId }`.

`visibility-orchestrator.ts:274+` `createRun()` writes an `OrchestratorRun(status=PENDING)` and returns immediately. The route then enqueues a BullMQ job onto `orchestratorQueue`.

`backend/src/workers/orchestrator.worker.ts`:
- Concurrency = **4** (bumped from 2 in `88f2a79`).
- Wraps the job in `runWithUsageContext()` so all token usage attributes to the right `(companyId, feature)`.
- Daily autopilot job is registered via `scheduleDailyAutopilot()` (cron `0 6 * * *`, **disabled by default** — set `AUTOPILOT_CRON_ENABLED=true` to enable).

`OrchestratorRun.status` transitions: `PENDING → PLANNING → EXECUTING → COMPLETED | FAILED | CANCELLED`.

### 5.2 Plan-mode loop

`visibility-orchestrator.ts:287+`. Constants:
- `MODEL = "claude-sonnet-4-6"` (Haiku dropped proposals to zero on 2026-04-28).
- `MAX_TURNS = 10` (real runs converge in 3–4).
- `RUN_TIMEOUT_MS = 180_000` (180s wall-clock cap; bumped from 90s).
- `max_tokens = 16_000` per Anthropic call.
- System prompt is cached as an ephemeral block — same prompt across all turns within the 5-minute TTL.

The loop:

```
budgetGuard.check()                    // daily caps: 2M tokens, $5 USD
for turn in 0..MAX_TURNS:
  if Date.now() - startedAt > RUN_TIMEOUT_MS:
    if proposals.length == 0: mark TIMED_OUT
    else: log "Plan ready — N proposals"; break
  resp = anthropic.messages.create({ system, tools, messages, max_tokens: 16_000 })
  for tool_use in resp.content:
    switch tool.name:
      case 'finish':         summary = ...; break loop
      case 'propose_plays':  proposals += validateAndCritique(plays)
      default:               handlers[tool.name](tool.input)
    push tool_result onto messages
end
write proposals → OrchestratorRun.plan
emit ActionEvent('plan_completed') per actionId
trajectoryLogger.finishRun(COMPLETED, summary)
```

### 5.3 The 10 tools

`backend/src/agents/tools/schemas.ts` defines them; `handlers.ts` implements them.

| Tool                    | Reads / Writes                                                            |
|--------------------------|---------------------------------------------------------------------------|
| `list_actions`           | open Actions for the company (capped at 30, sorted by priority)           |
| `get_evidence`           | one Action + affected prompts + sim excerpts + per-prompt citation sources; if surface_scout, also `scoutCandidates`/`scoutSummary`/`scoutDomain` |
| `get_citations`          | uncapped citation source list, optional bucket/minCount filters           |
| `get_competitor_presence`| domains → which competitors cited there                                   |
| `get_past_plays`         | recent ContentDraft rows (cap 20) — lets the model avoid repetition       |
| `get_learnings`          | last 10 `OrchestratorLearning` rows for the company                       |
| `propose_plays`          | submits proposals for validation (plan-mode only)                         |
| `generate_play`          | **disabled in plan mode** — calls writer + critic loop in execute mode    |
| `write_memory`           | inserts into `OrchestratorLearning`                                       |
| `finish`                 | break the loop, persist summary                                           |

### 5.4 Sub-agents

All in `backend/src/agents/sub-agents/`.

**research.ts** — pure data layer (no LLM). `listActions`, `getEvidence`, `getCitations`, `getActionBucket`, `getSourcesForPrompt`, `getCompetitorPresence`, `getPastPlays`. Joins are minimised (one DB hit, group in JS).

**planner.ts** — thin wrapper around `contentLabService.suggestPlays()`. Returns ranked plays for a single action.

**content-writer.ts** — adapter over `contentLabService.generateDraft()`. Preserves the LinkedIn / Reddit / Twitter / Quora / blog / email prompts byte-for-byte (see comment at top of `sub-agents/README.md`).

**critic.ts** — Sonnet judge. System prompt: *"Grade on evidence alignment, platform fit, concrete facts, citation-worthiness, brand safety."* Forced tool-use: `submit_score`. Returns `{ score: 0..1, accept: score ≥ 0.6, reasons: string[] }`. Threshold `CRITIC_THRESHOLD = 0.6`.

**surface-scout.ts** — already covered in §3.8.

### 5.5 Budget guard

`backend/src/agents/runtime/budget-guard.ts`:
```
DEFAULT_DAILY_TOKEN_CAP = 2_000_000
DEFAULT_DAILY_USD_CAP   = 5.00
```
`check()` sums `OrchestratorRun.tokensUsed / costUsd` for runs created since `00:00 today` and returns `{ tokensUsedToday, costUsdToday, tokensRemaining, usdRemaining, allowed }`. Called at `runPlan` start.

### 5.6 Trajectory logger

`backend/src/agents/runtime/trajectory-logger.ts` writes `OrchestratorStep` rows. Step kinds: `PLAN`, `TOOL_CALL`, `SUB_AGENT`, `CRITIQUE`, `OBSERVATION`, `RETRY`, `FINAL`. Per-tool input/output and token counts are persisted, which is what powers the live tool-call indicator in §2.4 and the trajectory drawer in MissionControl (§2.7).

### 5.7 Reaper

`backend/src/workers/orchestrator-reaper.ts` runs every 60s. Any run still in `PENDING|PLANNING|EXECUTING` with `startedAt < now() - 420_000ms` (= `2 × RUN_TIMEOUT + 60s safety`) is force-terminated to `FAILED` with reason `"zombie reaped — no terminal state written within 7m"`. It does NOT cancel the in-flight Anthropic call — just marks the row.

---

## 6. Proposal validation

`backend/src/agents/orchestrator/proposal-validator.ts`. Triggered by every `propose_plays` call.

### 6.1 The 8 passes

1. **Empty guard** (caller).
2. **Sort** by `confidence` DESC.
3. **playType ∈ VALID_PLAY_TYPES** (26 types — see `proposal-validator.ts:49–71`).
4. **PER_ACTION_CAP = 8** — at most 8 proposals per actionId.
5. **Deterministic competitor blocklist** — for PITCH plays, reject if `isCompetitorDomain(channelKey, competitorBlock)`. The blocklist is built from `Competitor` rows and brandHub competitors in `analyzedData`, with name → slug normalisation.
6. **Global outlet × playType dedupe** — one (outlet, playType) pair max across the run.
7. **Confidence clamp + channel enrichment** — `enrichWithChannel()` populates `channelKey / channelLabel / channelAction`.
8. **Critic LLM domain classifier** — every PITCH-family domain is sent to a Haiku judge (`critiqueProposalDomains`, `:190–257`) for `classification ∈ {client, competitor, neutral}` and `industryFit ∈ {fit, adjacent, mismatch}`. Reject if `competitor` OR `mismatch`.

### 6.2 Surface-scout anchoring (Phase 5.1)

`isAnchoredToScout()` (`:319–357`) preloads a `scoutDomainByAction` map from any `evidenceType=surface_scout` actions. A proposal targeting a surface-scout action must point at:
- (a) the scoutDomain itself, OR
- (b) a client-owned domain, OR
- (c) one of `ANCHOR_COMMUNITY_DOMAINS` = `{reddit.com, quora.com, linkedin.com, x.com, twitter.com, medium.com}`,

else it's rejected as `unanchored`.

### 6.3 PITCH play types

`PITCH_PLAY_TYPES` (`:37–45`): `email_pitch, guest_post_pitch, listicle_inclusion_pitch, product_listing_request, partnership_inquiry, editorial_tip, press_release`.

---

## 7. Content drafting (per play type)

### 7.1 Entry

`backend/src/services/content-lab.service.ts:21+` exports `generateDraft({ companyId, contentNeedId, actionId?, playType?, targetChannel? })`. Called by `content-writer.ts → handlers.generate_play` in execute mode.

Returns `ContentDraft` row: `{ id, title, slug, contentType, playType, status: 'ready', markdown, html, jsonLd, metaTags, instructions, generatedAt }`.

### 7.2 PLAY_INSTRUCTIONS (per playType)

Each `playType` has a system-prompt template under `PLAY_INSTRUCTIONS` (`content-lab.service.ts:373+`). Excerpts:

| playType            | Length        | Required structure                                                                 |
|---------------------|---------------|-------------------------------------------------------------------------------------|
| `blog_post`         | 800–1500 w    | H2/H3, FAQ section (5+ Q&As), concrete facts, "Key Takeaways" section               |
| `pillar_page`       | 1500–2500 w   | Opening summary (NO `TL;DR` label), TOC, 6–10 H2 sections, FAQ 8–12 Q&As, schema     |
| `faq_page`          | 10+ Q&As      | 2–4 sentences each, JSON-LD `FAQPage` schema                                        |
| `comparison_article`| 800–1200 w    | Opening verdict, feature table, per-product pros/cons, "When to choose", FAQ        |
| `trust_page`        | 600–1000 w    | FAQ-first, opening (no hedging), 10–14 specific Q&As, 1 CTA, schema                 |
| `case_study`        | 600–800 w     | Challenge → Solution → Results, specific numbers, testimonial placeholder           |
| `product_page`      | 600–1000 w    | Hero, 3–5 value props, feature grid, social proof, pricing/CTA, FAQ                 |
| `lead_magnet`       | 11 sections   | Title promise, slug, persona, outline, 3 proof points, landing copy, schema         |
| `linkedin_post`     | 200–300 w     | First-person plural (BRAND), hook + insight, 3–5 hashtags + engagement question. NO mention of AI/SEO/competitors. |
| `reddit_response`   | 150–300 w     | Subreddit-aware, anchor a recent visit + small flaw, mention brand once, casual first-person, NO URLs, NO CTAs, NO em-dashes, NO AI-rhetoric reversal, honour wiki "frequently asked" rules |
| `twitter_thread`    | 5–7 tweets    | Each <280 chars, BRAND account (not fan persona), numbered `1/`, hashtags only on last tweet, NO `**bold**`, NO `TL;DR` |
| `quora_answer`      | 200–400 w     | Personal-experience anchor, mention brand naturally                                 |
| `email_pitch`       | <150 w        | Plain text, 2–3 targets, subject line (news angle), personalized opener referencing THEIR article, hook + 3 facts + CTA + website + sign-off. NO mention of AI/visibility/rankings. |
| `directory_listing` | 4 fields      | ≤160 char short desc, 250–400 char long desc, categories, pricing, 8–12 feature tags |

### 7.3 Reddit-specific behavior

- `parseSubredditFromChannel(targetChannel)` (`content-lab.service.ts:132–137`) extracts the bare subreddit name from `r/AskUK`, `reddit.com/r/AskUK`, etc.
- The orchestrator threads `targetChannel` through `executeProposals → generate_play → contentWriter.write → generateDraft` so the prompt has the right context.
- Reddit prompt instructions force a PRE-FLIGHT CHECK: FAQ-frequent topics, brand prominence, open-recommendation questions, URL/em-dash bans (commit `88f2a79`).
- Subreddit guesses include GB/UK regions (`AskUK`, `CasualUK`, `unitedkingdom`) when relevant.

### 7.4 Output sanitisation

Every draft passes through `sanitizeOutput()` (`content-lab.service.ts:260–371`):

- `stripMetaLeakFromLines` strips lines matching `META_LEAK_PATTERNS` (visibility/rankings/hedging language).
- `stripBrandUrls` removes the company's own website URL (especially in `reddit_response`).
- Em-dashes (`—`) → spaced hyphen.
- `**bold**` and `*italic*` removed for "humanVoice" plays.
- `AI_TELL_PHRASES` replacements for `reddit_response`:
  - `"billed as marketing fluff until"` → deleted
  - `"sounds like marketing speak until"` → deleted
  - `"doing things that shouldn't be possible"` → `"pulling off genuinely wild moves"`
  - `"cattle-herd situation"` → `"too packed"`
  - Marketing CTAs (`"check it out"`, `"give it a try"`, `"worth looking up"`) → deleted
- All "TL;DR" labels are stripped from generated content (memory rule, commit `88f2a79`).

### 7.5 Critic loop (execute mode)

`handlers.generate_play` (`backend/src/agents/tools/handlers.ts:99–158`) wraps the writer in a retry loop:

```
for attempt in 0..MAX_RETRIES (1):
  draft = contentWriter.write({ companyId, contentNeedId, actionId, playType, targetChannel })
  critique = critic.evaluate({ actionTitle, actionReason, impactArea, playType, draftTitle, draftMarkdown })
  trajectoryLogger.logStep(CRITIQUE, ...)
  if critique.accept (score >= 0.6) break
return { draftId, title, critique, attempts }
```

So every persisted draft has been judged by a separate Sonnet pass with the same evidence as the writer.

---

## 8. Lifecycle, events, and the closed-loop outcome worker

### 8.1 Models

- **`Action`** — see §4.1
- **`ActionEvent`** — funnel rows: `id, actionId, companyId, userId?, kind, metadata?, createdAt`. Indexed on `[actionId, createdAt]`, `[kind, createdAt]`, `[companyId, createdAt]`.
- **`ActionOutcome`** — closed-loop measurement: `id, actionId, completedAt, visBaseline, vis14d, vis30d, sentBaseline, sent30d, newCitationsHit, concurrentActions, metadata, computedAt`.
- **`OrchestratorRun`** — `status, trigger (MANUAL|POST_SIMULATION|CRON), plan (Json), summary, tokensUsed, costUsd, errorMessage, startedAt, completedAt, durationMs`.
- **`OrchestratorStep`** — per-tool / per-sub-agent step trace.
- **`OrchestratorLearning`** — distilled lessons used by `get_learnings`.
- **`ContentDraft`** — see §7.1.

### 8.2 Action funnel (kinds)

`backend/src/services/action-event.service.ts:23–33`:

```
KNOWN_KINDS = {
  opened, plan_started, plan_completed,
  draft_viewed, draft_copied, draft_approved, draft_rejected,
  action_completed, action_dismissed,
}
```

`recordActionEvent()` is best-effort: validates `kind`, looks up `companyId` if absent, inserts; on failure logs + returns `null` so the originating route still succeeds.

### 8.3 ActionOutcome worker

`backend/src/workers/action-outcome.worker.ts`:
- Runs nightly **02:30 UTC**, batches **50 actions** per sweep, polls every **6h**.
- For each action whose status flipped to `completed` and that has no `ActionOutcome` row yet:
  1. Resolve `metadata.affectedPrompts[].id`.
  2. Fetch `DailySnapshot` for the company across three windows (baseline = 7d before completion, +14d, +30d).
  3. Average visibility across the prompt intersection.
  4. Average company-level sentimentScore.
  5. Count new citations on `metadata.targetDomains[]` since `completedAt`.
  6. Count concurrent completed actions within ±7d on overlapping prompts.
  7. Insert `ActionOutcome`.

This is the dataset Phase 6 uses to *re-fit the magic weights* in Engine B (see §9.5).

---

## 9. Metrics, KPIs, and how they're computed

### 9.1 Hero counter

```
"<X> open issues, <Y> high priority"
```
Counted client-side from the `actions` query (`page.tsx:380`).

### 9.2 SummaryStrip — 4 KPI cards

| Card                 | Formula                                                                              | Source                          |
|----------------------|--------------------------------------------------------------------------------------|---------------------------------|
| **High-priority queue** | `actions.filter(a => a.priority==='HIGH' && a.status==='open').length`               | `page.tsx:380`                  |
| **Running boosts**      | `boosts.filter(b => b.status==='RUNNING').length`                                    | `page.tsx:381`                  |
| **Visibility uplift**   | `+min( actions.filter(a => a.status==='open' && (a.impactArea==='visibility' \|\| a.impactArea==='citations')).length × 3, 25 )%` | `page.tsx:344-345, 388` |
| **Recently resolved**   | actions with `status==='completed'` and `completedAt ≥ now-7d`                       | `page.tsx:382–386`              |

> ⚠️ The "Visibility uplift" tooltip in `metric-definitions.ts:318–322` advertises "Σ predicted ΔSoV per open action" but the implementation is the flat `count × 3% capped at 25%` heuristic above. The user-facing rationale and the code are out of sync. Fix one of them before quoting it externally.

### 9.3 Recommended-next banner

Sort open actions by `PRIORITY_RANK = { HIGH: 0, MEDIUM: 1, LOW: 2 }`, then `createdAt DESC`. Top entry renders as the banner above the queue. (`page.tsx:391–401`).

The j/k queue ordering uses `navOrderActions` (sorted by stage, then priority/createdAt) — *not* the recommended-next sort — so jumping with `j` does not always land on the highest-priority item.

### 9.4 In-action numbers (detail pane)

Most numbers shown in the detail pane come straight out of `metadata`:

- **Risk score / revenue score** = the Engine B integer described in §3.4.
- **Primary loss rate** = `topCompetitor.primaryWins / total` for the underlying topic cluster.
- **Citation power** = `citationCount` for the source.
- **Consensus score** = `mentionedModels.length / totalModels` for divergence actions.
- **Channel opportunity score** = `Σ (trustWeight × citationCount × lossRate) + (competitorHubCount × 3) + (hasMomentum ? 6 : 0)` across sources in the lane.

### 9.5 Why these magic numbers? (and how to defend them)

**`CATEGORY_WEIGHTS = { discovery: 1.0, comparison: 1.2, trust_validation: 1.5 }`** — round-number eyeballing of "early ↔ mid ↔ late funnel": a discovery-prompt loss is the baseline, comparison ~20% worse (further down funnel), trust ~50% worse (highest purchase intent).

**`× 1.6` per primary win** — owning the #1 slot should outweigh a generic loss but not dominate it. The author picked the midpoint of (1, 2): >1 because "primary > mention", <2 to avoid letting one strong competitor erase the broader signal.

**`× 0.35` per additional competitor** — crowded answers dilute everyone, so each extra competitor is a fractional penalty rather than a full loss-run. 0.35 happens to match `COMPETITOR_PRESENCE_SCORE.MENTION` on `:173`; same intuition: a non-primary mention is worth ~⅓ of a primary slot.

**Bottom line:** these are seat-of-the-pants weights chosen so HIGH/MEDIUM bucket sizes feel right on the dev tenants. The thresholds (45/40/0.6/0.55) were calibrated *to* the weights, not derived from outcomes. Until `ActionOutcome` accumulates enough rows to fit them by Lasso, treat the score as a sortable risk index — not a prediction.

There is a calibration script at `backend/scripts/calibrate-action-weights.ts` that runs the full Engine B math on a single company and stress-tests the constants — useful before any tweak.

---

## 10. API endpoints called by the page

### 10.1 Reads

| Endpoint                                                              | Used by                                          |
|------------------------------------------------------------------------|--------------------------------------------------|
| `GET /beacon/company/:id/actions?status=all&timeRange&model`           | queue + KPIs                                     |
| `GET /boosts/company/:id`                                              | boost list                                       |
| `GET /content-lab/company/:id/platforms`                               | recommended pitch outlets                        |
| `GET /content-lab/company/:id/orchestrator-runs`                       | proposals + run history (MissionControl, panel)  |
| `GET /content-lab/company/:id/drafts`                                  | persisted drafts                                 |
| `GET /beacon/company/:id/{trust\|comparison\|discovery}`               | evidence prompts in detail pane                  |
| `GET /content-lab/orchestrator-run/:id` (1.5s polling)                 | live tool-call indicator                         |
| `GET /content-lab/orchestrator-run/:id` (4s polling)                   | execute progress                                 |

### 10.2 Writes

| Endpoint                                                              | Trigger                                             |
|------------------------------------------------------------------------|------------------------------------------------------|
| `POST /beacon/company/:id/actions/generate`                           | "Scan for actions" CTA                              |
| `PATCH /beacon/company/:id/actions/:actionId`                         | status dropdown (open / completed / dismissed)      |
| `POST /beacon/company/:id/action-event`                               | every `recordActionEvent` call                      |
| `POST /content-lab/company/:id/orchestrate`                           | "Run Autopilot", or auto-trigger on action open (mode=plan, focusActionId) |
| `POST /content-lab/orchestrator-run/:id/execute`                      | "Generate content" after selecting proposals        |
| `POST /content-lab/orchestrator-run/:id/cancel`                       | MissionControl cancel                               |
| `POST /boosts/company/:id/from-action`                                | "Create workflow" from an action                    |
| `POST /boosts/company/:id/from-goal`                                  | custom workflow input                               |
| `POST /boosts/:boostId/start`                                         | "Launch" on an orphan boost                         |
| `POST /boosts/steps/:stepId/approve`                                  | step approval                                       |
| `POST /boosts/steps/:stepId/skip`                                     | step skip                                           |

---

## 11. Knobs, env vars, and queue topology

### 11.1 Env vars

| Name                              | Default                  | Effect                                                                |
|-----------------------------------|--------------------------|-----------------------------------------------------------------------|
| `ORCHESTRATOR_MODEL`              | `claude-sonnet-4-6`      | Plan-loop model. Haiku is known to under-produce proposals.           |
| `ORCHESTRATOR_MAX_TURNS`          | `10`                     | Hard cap on tool-use turns per run.                                   |
| `ORCHESTRATOR_TIMEOUT_MS`         | `180000`                 | Wall-clock cap; reaper threshold = `2× + 60s` = `420000ms`.            |
| `ORCHESTRATOR_MAX_OUTPUT_TOKENS`  | `16000`                  | Per-call max_tokens.                                                  |
| `AUTOPILOT_CRON_ENABLED`          | `false`                  | Daily orchestrator sweep at `0 6 * * *` UTC.                          |
| `ORCHESTRATOR_DAILY_TOKEN_CAP`    | `2000000`                | Budget guard, per-day per-tenant.                                     |
| `ORCHESTRATOR_DAILY_USD_CAP`      | `5.0`                    | Same.                                                                 |

### 11.2 Worker topology

```
orchestratorQueue (BullMQ over Redis)
  ├─ concurrency 4 (was 2)
  ├─ daily cron job (06:00 UTC, fan-out per company)
  └─ jobs: { companyId, trigger, mode, runId, selections, focusActionId }

orchestrator-reaper
  ├─ setInterval(60_000)
  └─ marks stale runs FAILED (no Anthropic cancellation)

action-outcome.worker
  ├─ nightly 02:30 UTC, polls every 6h
  └─ batch 50, fills ActionOutcome rows
```

### 11.3 Anti-drift / cost levers

- System prompt cached as ephemeral block — same prompt across all turns within 5 min TTL → near-zero re-tokenisation.
- Tools list cached the same way.
- Token usage logged per tool call (input / output / cache_read / cache_create) and rolled up onto `OrchestratorRun.tokensUsed / costUsd`.

---

## 12. Failure modes and where to look first

| Symptom                                              | Likely cause / where to look                                                                                       |
|------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------|
| "No proposals" after a plan run                      | Critic rejected everything. Look at `OrchestratorStep.kind=CRITIQUE` rows for the run.                              |
| Plan keeps showing `running…` past 7 minutes         | Reaper hasn't fired. Check `orchestrator-reaper` is registered at boot and Redis is reachable.                      |
| Same outlet pitched twice from one run               | Validator dedupe pass 6 didn't see them as the same outlet — usually `normalizeHost` mismatch (`www.` vs bare).     |
| HIGH count on the strip ≠ what you see in the queue  | Filter mismatch: KPI counts `status === 'open'` regardless of stage; queue may have moved the action to `Running`.  |
| Drafts contain "—" or `TL;DR`                        | `sanitizeOutput()` not called or fragments slipped past it. Re-check `META_LEAK_PATTERNS` and `AI_TELL_PHRASES`.    |
| Auto-trigger keeps re-firing on action open          | 6h cooldown stamp missing on `OrchestratorRun.metadata` or `LastTriggerAt` not persisted.                            |
| Cards show 0 prompts/sims for a company that ran     | Wrong route: `GET /companies` was using unfiltered `_count`. Fixed to `prompts: { where: { isActive: true } }`.     |
| Surface-scout actions with no candidates             | Page fetch failed or LLM returned non-JSON. `surface-scout.ts:165–231` swallows per-source errors, check logs.       |
| ActionOutcome rows missing                           | `companyId.affectedPrompts` was empty → outcome worker skips. Verify Engine B set `metadata.affectedPrompts`.       |

---

### Appendix A — One-screen file index

```
backend/src/services/action-intelligence.service.ts   ~2030L  Engine B (lanes, scoring, dedup)
backend/src/routes/beacon.routes.ts                          /actions GET + /actions/generate
backend/src/services/action-event.service.ts                 ActionEvent insert + validation
backend/src/services/content-lab.service.ts                  generateDraft + PLAY_INSTRUCTIONS
backend/src/agents/orchestrator/visibility-orchestrator.ts   Plan loop (Sonnet, MAX_TURNS=10)
backend/src/agents/orchestrator/proposal-validator.ts        8-pass validator + Haiku domain critic
backend/src/agents/sub-agents/research.ts                    Pure-data tool implementations
backend/src/agents/sub-agents/planner.ts                     suggestPlays adapter
backend/src/agents/sub-agents/content-writer.ts              generateDraft adapter
backend/src/agents/sub-agents/critic.ts                      Sonnet judge (CRITIC_THRESHOLD=0.6)
backend/src/agents/sub-agents/surface-scout.ts               Long-tail scout (Haiku + Sonnet)
backend/src/agents/tools/{schemas,handlers,index}.ts         10 tools (list_actions, propose_plays, ...)
backend/src/agents/runtime/budget-guard.ts                   2M tokens / $5 daily caps
backend/src/agents/runtime/trajectory-logger.ts              OrchestratorStep writes
backend/src/agents/memory/learning-store.ts                  OrchestratorLearning store
backend/src/workers/orchestrator.worker.ts                   BullMQ worker (concurrency 4)
backend/src/workers/orchestrator-reaper.ts                   Zombie reaper (60s, 7m threshold)
backend/src/workers/action-outcome.worker.ts                 Closed-loop outcome computation

frontend/app/companies/[id]/(beacon)/actions/page.tsx        CommandCenterPage (~1077L)
frontend/components/beacon/action-detail-pane.tsx            Detail pane
frontend/components/beacon/action-queue.tsx                  9-stage queue, j/k nav
frontend/components/beacon/play-cards.tsx                    PlayDraftRenderer + per-platform cards
frontend/components/beacon/mission-control.tsx               In-flight run pill + Run Autopilot modal
frontend/components/beacon/right-drawer.tsx                  Mobile/tablet drawer
frontend/components/beacon/proposal-list.tsx                 PerActionProposals
frontend/components/beacon/boost-detail-pane.tsx             Boost / custom workflow pane
frontend/components/beacon/pitch-detail-pane.tsx             Pitch outlet pane
frontend/lib/action-events.ts                                recordActionEvent
frontend/lib/metric-definitions.ts                           Tooltip copy for KPIs
frontend/stores/beacon-filters.ts                            timeRange + model zustand store
```

### Appendix B — End-to-end sequence diagram

```
User opens Action ────────────────────────────────────────────────────────────▶
                                                                              │
                       page.tsx setSelectedKey()                              │
                       └─ recordActionEvent('opened')                         │
                       └─ if not on cooldown: POST /content-lab/.../orchestrate │
                                  │                                            │
                                  ▼                                            │
                  beacon route: visibility-orchestrator.createRun() ──▶ DB     │
                                  │                                            │
                                  ▼                                            │
                  orchestratorQueue.add()  ──BullMQ──▶ orchestrator.worker     │
                                                                ▼              │
                                                visibility-orchestrator.runPlan│
                                                    ├─ budgetGuard.check       │
                                                    ├─ for turn in 0..10:      │
                                                    │   anthropic.messages    │
                                                    │   for tool_use:          │
                                                    │     handlers[tool]       │
                                                    │     proposal-validator   │
                                                    │   end                    │
                                                    └─ trajectoryLogger.finish │
                                                                ▼              │
                                                Update OrchestratorRun.plan ──▶│
                                                                               │
                       page polls /orchestrator-runs (3s) ◀───────────────────│
                       proposals appear in detail pane                         │
                       User picks subset, clicks "Generate"                    │
                                  │                                            │
                                  ▼                                            │
                  POST /orchestrator-run/:id/execute                           │
                                  │                                            │
                                  ▼                                            │
                  worker picks up, mode=execute                                │
                       for each Proposal:                                      │
                         handlers.generate_play                                │
                           ├─ contentWriter.write → contentLab.generateDraft   │
                           │     (Sonnet, PLAY_INSTRUCTIONS by playType)       │
                           ├─ sanitizeOutput()                                 │
                           ├─ critic.evaluate (Sonnet, score≥0.6)              │
                           └─ retry once if rejected                           │
                                  │                                            │
                                  ▼                                            │
                  ContentDraft rows persisted ──▶ ActionEvent('plan_completed') │
                                                                               │
                       page polls /drafts (30s)                                 │
                       PlayDraftRenderer routes by playType                    │
                       User views/copies → recordActionEvent('draft_viewed' / 'draft_copied')
                       User marks status='completed' → recordActionEvent('action_completed')
                                                                               │
                       Nightly 02:30 UTC: action-outcome.worker computes      │
                       baseline / +14d / +30d visibility deltas → ActionOutcome│
```
