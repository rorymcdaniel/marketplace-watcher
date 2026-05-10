# Facebook Marketplace Watcher — Product Requirements Document

**Version**: 0.1 (draft)
**Date**: 2026-05-10
**Status**: For review
**Owner**: Rory

---

## 1. Problem & Vision

The user enjoys finding bargains on Facebook Marketplace but does not want to spend time *on* Facebook Marketplace. Marketplace is engineered to keep users on the site — endless feed, irrelevant suggestions, inconsistent filters that silently drop matching items — and that loop is precisely what the user wants to avoid.

The Marketplace Watcher is a personal automation that runs saved searches on a schedule, applies stricter and more reliable filtering than Marketplace itself, scores every examined listing against the user's criteria, and pushes only high-confidence matches to a Discord channel. The user evaluates each notification and clicks through to Marketplace *only* for confirmed-interesting items — never to browse.

The product's adversary is the user being pulled back into Marketplace browsing. Every design decision should be evaluated against that.

This is intentionally **not** an AI-powered product. Scoring and filtering are deterministic, rule-based, and explainable. Any AI/ML judgment about whether a listing is "good" is explicitly out of scope.

## 2. Goals

- Run scheduled searches against Marketplace without human intervention.
- Apply deterministic, configurable filtering that is stricter and more reliable than Marketplace's own.
- Score every examined item against the search's criteria; surface only items meeting or exceeding a configurable confidence threshold.
- Never re-surface an item the user has already been notified about or that has been recorded as rejected.
- Make notifications self-contained enough that the user can decide *without opening Marketplace* in the common case.
- Provide rich, structured logging and error reporting that is queryable by both humans and AI agents.
- Keep all interfaces — notification dispatcher, search definitions, CLI — abstract and stable enough that future additions (interactive triage, an AI agent that adds searches on the user's behalf, additional notification channels) bolt on cleanly without rework.

## 3. Non-Goals (v1)

- **No AI/ML judgment on listings.** Scoring is a deterministic weighted formula. Image content is not analyzed. Description text is not summarized or interpreted. The product does not use a language model to decide whether a listing is "a good deal."
- **No interactive triage / feedback loop in v1.** The notification interface must accommodate it later, but v1 is fire-and-forget.
- **No agent integration in v1.** A future "Claude, add this to my saved searches" or "Claude, run this search now" capability is anticipated, but not built. The CLI/config surface should not foreclose it.
- **No seller blocking or seller-reputation scoring.**
- **No cross-platform search** (Craigslist, OfferUp, etc.).
- **No multi-user support, no auth model.**
- **No mobile app or rich web UI.** CLI plus a config file is acceptable.
- **No write actions on Marketplace.** No messaging sellers, no offers, no saves.
- **No re-scoring of historical items** when criteria change. New criteria apply to future runs only.
- **No backfill of missed runs** when a paused search resumes or after downtime.

## 4. Anti-Goals

These are things the product must actively avoid, distinct from features it merely doesn't have:

- **Do not pull the user back into Marketplace browsing.** Notifications must link to a *specific listing*, never to search results, category pages, or any surface that invites scrolling. Notification payloads should be rich enough that the user can usually skip opening Marketplace at all.
- **Do not over-notify.** A false positive (notification for an item the user wouldn't have wanted) is more costly than a false negative (a real bargain missed). Defaults should reflect this.
- **Do not introduce non-determinism.** Given the same listing data and criteria, scoring must produce the same output every time. No randomness, no model calls, no time-of-day variation.

## 5. User Stories

- **U1**: As the user, I define a set of saved searches and schedules, and the system runs them without my involvement.
- **U2**: As the user, I receive a Discord message *only* when an item scores at or above the search's threshold.
- **U3**: As the user, I can usually decide whether to pursue a listing from the Discord message alone, without opening Marketplace.
- **U4**: As the user, I never see the same listing twice in notifications.
- **U5**: As the user, I can pause any search without deleting it, and resume it later with full state intact.
- **U6**: As the user, I can inspect why the system did or did not surface a given item — every score component is explainable.
- **U7**: As the user (or as Claude on the user's behalf), I can investigate failures and unusual behavior by querying structured logs and error reports.
- **U8**: As the user, I can run a "test" of a saved search end-to-end without sending notifications, to validate criteria changes.

## 6. Functional Requirements

### 6.1 Search Definitions

Each saved search specifies:

| Field | Type | Required | Notes |
|---|---|---|---|
| `id` / `name` | string | yes | Stable identifier; appears in notifications. |
| `query` | string | yes | The text query submitted to Marketplace. |
| `origin_zip` | string | yes | For distance calculations. |
| `max_distance_miles` | number | yes | Hard filter. |
| `max_price` | number | yes | Hard filter. |
| `min_price` | number | no | Defends against "$1 to get clicks" listings. |
| `result_cap_per_run` | integer | no, default **50** | Max results examined per run. |
| `keywords_must_contain_any` | list of strings | no | At least one must hit (title or description). |
| `keywords_must_not_contain` | list of strings | no | Any hit disqualifies the listing. |
| `keywords_boost` | list of strings | no | Soft signal — hits raise score, misses don't disqualify. |
| `score_weights` | object | no | Per-search overrides of global weights (see 6.4). |
| `threshold` | number 0–100 | no, default **75** | Score at or above this triggers a notification. |
| `schedule` | cron expression | yes | Resolved by the external system cron. |
| `paused` | boolean | no, default false | Scheduler skips paused searches; state preserved. |
| `created_at` / `updated_at` | timestamp | system | For audit. |

**Symmetry rule**: every text-matching criterion supports both `contains` and `does_not_contain` forms. Numeric and geographic criteria support only the form that makes physical sense.

### 6.2 Scheduling

- Searches are triggered by an external system cron. The product exposes a single entry point that reads the search config, determines which searches are due, and runs them.
- Multiple searches due at the same tick run sequentially in v1.
- Each run produces a run record with status, counts, timestamps, and a correlation handle that ties to its log stream.

### 6.3 Result Fetching

- The system fetches up to `result_cap_per_run` listings for the search query.
- For each listing, the system extracts: listing ID, title, description, price, location (lat/lon or city/zip), posted timestamp, image count, first image URL (for embed thumbnails), seller display name, listing URL.
- Listings missing required fields (e.g., no price) are recorded as "skipped — incomplete data" with the specific missing field, and not scored. Logged at info.

### 6.4 Filtering & Scoring

**Hard filters** (failure on any drops the listing before scoring):
- Price outside `[min_price, max_price]`.
- Distance from `origin_zip` exceeds `max_distance_miles`.
- Any `keywords_must_not_contain` term appears in title or description (case-insensitive, word-boundary-aware).
- No `keywords_must_contain_any` term appears in title or description (when this list is non-empty).

**Score components** (each normalized to 0–100, then combined via weighted average):

| Component | Description |
|---|---|
| **Keyword strength** | Combines (a) density of must-contain + boost keyword hits, (b) title-vs-description weighting (title hits worth more), (c) a multi-keyword bonus that scales nonlinearly so "all keywords matched" is meaningfully better than "one matched." |
| **Price headroom** | How far below `max_price` the listing is. Lower price → higher score. |
| **Distance** | Closer to `origin_zip` → higher score. Floors at `max_distance_miles`. |
| **Freshness** | Newer listings score higher. Listings older than 30 days hard-floor at 0. |
| **Image presence** | 0 images → 0; 3+ images saturates. |

**Default global weights** (sum to 1.0):

| Component | Weight |
|---|---|
| Keyword strength | 0.35 |
| Price headroom | 0.25 |
| Distance | 0.20 |
| Freshness | 0.12 |
| Image presence | 0.08 |

These defaults are conservative — keyword match and price dominate, soft signals are minor — consistent with the anti-goal of over-notifying.

Each search may override any subset of weights via `score_weights`. Overrides are normalized so the active weights still sum to 1.0.

**Default threshold**: 75. Set high deliberately. Easier to relax than to recover from notification fatigue.

**Explainability requirement**: every scored item has a stored breakdown of:
- Which hard filters it passed (or which one it failed and was dropped on).
- Per-component sub-scores (0–100).
- Which keywords hit, and where (title vs. description).
- The active weights used (post-override, post-normalization).
- The final composite score.
- The threshold it was compared against.
- Whether it was notified, and when.

### 6.5 Deduplication & State

- Every examined listing is recorded by Marketplace listing ID.
- A listing ID seen before is not re-scored and not re-notified.
- Once a listing has been examined — whether notified or not — it is considered "seen" and never resurfaces.
- v1 dedup is by listing ID only. Fingerprint-based dedup is deferred unless re-listing churn proves problematic in practice.
- Scored item records are retained for **6 months**, then pruned.
- Run records and aggregate statistics are retained for **12 months**.

### 6.6 Notifications (Fire-and-Forget v1)

**Channel**: Discord. The Discord bot itself is provided externally (out of scope to create). The product needs a configurable webhook URL or bot token plus channel ID.

**v1 routing**: a single Discord channel receives notifications for all searches. The originating search name is included in every message. Per-search channel routing is a future enhancement; the dispatcher interface should not preclude it.

**Notification payload** (Discord embed-shaped, but the *interface* is channel-agnostic):
- Search name (which saved search produced this).
- Listing title.
- Price.
- Distance from origin zip.
- Composite score and threshold.
- Top 1–3 matched keywords and where they hit.
- Posted-ago timestamp.
- First image as thumbnail (when available).
- Direct link to the specific listing.

The payload is intentionally rich so the user can decide without opening Marketplace.

**Interface design**: notifications are dispatched by passing a structured `NotifiableItem` to a notification dispatcher abstraction. The dispatcher chooses channels based on configuration. v1 ships one dispatcher (Discord). A future interactive-triage channel — e.g., Discord buttons that record approve/reject — plugs in without changing the scorer or the item store.

**Delivery accounting**: a successful send is recorded on the item record with timestamp and channel. Failed sends retry with exponential backoff (sane caps); after exhaustion, the failure is persisted to a dead-letter store and surfaced as an error event.

### 6.7 Pause / Resume

- Any search can be paused. Paused searches: no fetches, no scoring, no notifications, no state changes.
- Resuming a paused search produces no backfill — it simply runs at its next scheduled tick.
- Pause/resume actions are logged.

### 6.8 Configuration Changes

- Edits to a search's criteria take effect on the next scheduled run.
- Historical scored items are not re-scored against new criteria. (Re-scoring is out of scope.)
- Edits are logged with a before/after diff for auditability.

### 6.9 Test Run

- A CLI flag (e.g., `--dry-run` or `--test`) runs a search end-to-end *without* sending notifications and *without* recording examined listing IDs into the dedup store.
- Output reports what *would* have happened: counts of hard-filtered, scored, notified items, plus the full score breakdown for each item that would have been notified.
- This is the primary tool for validating criteria changes.

### 6.10 Logging & Observability

A v1 product requirement, not a stretch goal.

**Vendors**: deferred. Selection criteria: generous free tier suitable for personal-scale traffic; structured-log ingest; query API; stable; minimal vendor lock. Likely candidates include a logs service and a separate error-reporting service, but the choice is a later spike.

**Functional requirements**:
- Structured events (key/value) for all operational logs.
- Errors and exceptions reported to a centralized error-reporting service with full context: search id, run id, the input that triggered the failure, stack trace.
- Both services must expose a query API. AI agents (Claude in particular) must be able to investigate incidents programmatically, without screen-scraping a UI.
- Every log line includes `run_id`, `search_id` (where applicable), and a `correlation_id` that ties together everything from a single fetched listing through its score, dedup decision, and notification outcome.
- A per-run summary log records: searches run, items examined, items hard-filtered (with reason breakdown), items scored, items notified, errors raised. This is the "did anything weird happen overnight?" surface.

**Health and anomaly surface** (informational, not blocking):
- Per-search: last successful run, last failed run, consecutive failure count, last notification sent.
- Anomalies: a search that returned zero results when it historically returned dozens; a search that examined N results and hard-filtered all of them; sustained latency increase.
- "System alert" notifications for hard failures (e.g., Marketplace blocking, repeated scrape failures) sent through the same Discord channel as match notifications, but visually distinct (different message type, distinct embed color).

### 6.11 Failure Behavior

- **Zero results returned**: recorded as a non-error "empty result" event. Two consecutive empty runs for a search that was previously productive raises a health flag.
- **Distance not computable** (no listing location): listing is hard-filtered with reason "missing location," not scored. Logged at info.
- **All N results hard-filtered**: not an error; logged distinctly so a low-quality search definition is visible.
- **Marketplace blocks or page-shape change**: error reported with full context; run aborted (no partial commit); health surface reflects the failure; user notified via Discord with a "system alert" message type.
- **Notification delivery failure**: retry with exponential backoff; persist to dead-letter after exhaustion; surface as error event.

## 7. Non-Functional Requirements

- **Single-user**: no auth model in v1.
- **Idempotency**: a run is safe to retry. Either the run commits state atomically, or partial commits are explicit and resumable. A re-run after a mid-run crash must not produce duplicate notifications or duplicate "seen" entries.
- **Determinism**: same listing data + same criteria → same score, every time.
- **Configurability**: every numeric threshold, weight, cap, and retention window is configurable without code changes.
- **Auditability**: every notification can be traced to the exact criteria, weights, and listing snapshot that produced it.
- **Interface stability for future agent integration**: the CLI / config surface that runs searches and edits the search list must be scriptable enough that, later, an external AI agent can call it (a) to run a search on demand and (b) to add a new saved search. Building that integration is out of scope; not foreclosing it is a design constraint.

## 8. Out of Scope (v1, with extensibility hooks)

| Deferred feature | Hook in v1 |
|---|---|
| Interactive triage / approve-reject feedback | Notification dispatcher takes a structured `NotifiableItem`; new channels plug in without touching scorer or store. |
| Agent-driven search execution and search authoring | CLI and config surface are scriptable; no human-only assumptions in the interface. |
| Per-search Discord channel routing | Dispatcher is configuration-driven; routing logic added later without scorer/store changes. |
| Additional notification channels (email, push, SMS) | Dispatcher abstraction is channel-agnostic. |
| Seller blocking | Item record stores seller display name; can be filtered on later. |
| Fingerprint-based dedup | Item record stores enough fields (title, price, seller) that a fingerprint pass can be added later. |
| Re-score on criteria change | Not engineered for in v1. Add only if needed. |
| Concurrent search runs | Scheduler dispatches one at a time; trivially parallelizable later. |
| Price-drop re-notification | Out of scope in v1; possible to add by relaxing the dedup rule later. |

## 9. Success Criteria

- The user receives a Discord notification for any clearly-matching listing they would have wanted to see, given the criteria.
- Notification false-positive rate is low enough that the user does not develop notification fatigue. Measurable informally post-launch via user disposition; replaced by interactive-triage data once that ships.
- The user reports they spend meaningfully less time on Marketplace than before, and that the time they do spend is on confirmed-interesting listings reached via notification deep links — not browsing.
- No item is notified twice.
- The user (or Claude, on the user's behalf) can answer "why did this item get notified?" or "why did the 9pm run fail?" purely by querying logs and stored records.

## 10. Open Questions

1. Logging vendor and error-reporting vendor selection (deferred to a later spike; criteria specified in §6.10).
2. Notification config model — webhook URL vs. bot token + channel ID, where the secret lives, and how it's loaded.
3. Run record retention beyond 12 months — keep summary stats forever, or also prune?
4. Exact backoff parameters for notification delivery retries.
5. Distance computation precision — straight-line from zip centroid is the v1 default; is that good enough, or does the user want driving distance?
