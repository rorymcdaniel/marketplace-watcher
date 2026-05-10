# Facebook Marketplace Watcher — Implementation Plan

## Context

The user wrote `PRD.md` defining a personal automation that runs saved Facebook Marketplace searches on a schedule, applies stricter deterministic filtering than Marketplace itself, scores listings against per-search criteria, and pushes only high-confidence matches to a Discord channel. The product's adversary is the user being pulled back into Marketplace browsing — every interaction is fire-and-forget, payloads are rich enough to decide without opening Marketplace, and the user only deep-links to *specific listings*.

The codebase is empty (`/home/rory/Code/marketplace-watcher/PRD.md` is the only file). This plan greenfields the product from the PRD with all decisions made by the planner so the user can hand it to Sonnet and walk away.

The plan is structured in **13 phases**, each scoped so that a single subagent can complete it semi-independently with clear inputs from the prior phase and clear outputs the next phase consumes. Phases 0–7 are the build path; 8–12 round out observability, CLI surface, and tests; 13 is documentation. Sonnet should execute them in order. Phases marked "parallelizable" can be safely run by separate subagents simultaneously.

---

## Top-level decisions (made under ambiguity — reasoning at end)

| Decision | Choice |
|---|---|
| Language / runtime | **Python 3.11+** |
| Marketplace fetch strategy | **Playwright (headless Chromium) with stealth** |
| Storage | **SQLite** (single file, WAL mode) |
| Config format | **YAML** (per-user, single file) |
| CLI framework | **typer** (Click-based, scriptable) |
| Logging | **structlog → JSON lines** locally + Sentry for errors (vendor-pluggable) |
| Geocoding | **pgeocode** (offline ZIP→lat/lon) + **haversine** for distance |
| Schedule resolution | **croniter** (cron expressions in config; external cron drives one entrypoint) |
| Notification channel v1 | **Discord webhook** (URL in config / env var) |
| Test framework | **pytest + pytest-asyncio** |
| Packaging | **pyproject.toml** (PEP 621), `pip install -e .` workflow, `uv` compatible |
| Distribution | Local install + Docker Compose service for the cron host |

---

## Project layout

```
marketplace-watcher/
├── PRD.md                                # existing
├── README.md                             # phase 13
├── pyproject.toml
├── docker-compose.yml                    # phase 13
├── Dockerfile                            # phase 13
├── config.example.yaml                   # phase 13
├── .gitignore
├── src/marketplace_watcher/
│   ├── __init__.py
│   ├── __main__.py                       # `python -m marketplace_watcher`
│   ├── cli.py                            # phase 9 — typer entrypoint
│   ├── config/
│   │   ├── __init__.py
│   │   ├── schema.py                     # phase 2 — pydantic models
│   │   └── loader.py                     # phase 2
│   ├── store/
│   │   ├── __init__.py
│   │   ├── db.py                         # phase 1 — connection + WAL
│   │   ├── migrations.py                 # phase 1
│   │   ├── items.py                      # phase 1 — examined-item repo
│   │   ├── runs.py                       # phase 1 — run-record repo
│   │   └── health.py                     # phase 10
│   ├── fetch/
│   │   ├── __init__.py
│   │   ├── base.py                       # phase 3 — Fetcher protocol
│   │   ├── marketplace.py                # phase 3 — Playwright impl
│   │   └── extract.py                    # phase 3 — listing parser
│   ├── score/
│   │   ├── __init__.py
│   │   ├── filters.py                    # phase 4 — hard filters
│   │   ├── components.py                 # phase 4 — per-component scorers
│   │   ├── engine.py                     # phase 4 — composite + breakdown
│   │   └── geo.py                        # phase 4 — pgeocode wrapper
│   ├── notify/
│   │   ├── __init__.py
│   │   ├── base.py                       # phase 6 — NotificationDispatcher
│   │   ├── payload.py                    # phase 6 — NotifiableItem dataclass
│   │   ├── discord.py                    # phase 6 — Discord webhook impl
│   │   └── retry.py                      # phase 6 — backoff + dead letter
│   ├── orchestrator/
│   │   ├── __init__.py
│   │   ├── scheduler.py                  # phase 7 — due-search resolution
│   │   └── run.py                        # phase 7 — single-search executor
│   ├── obs/
│   │   ├── __init__.py
│   │   ├── logging.py                    # phase 8 — structlog setup
│   │   ├── errors.py                     # phase 8 — Sentry adapter
│   │   └── health.py                     # phase 10 — anomaly detector
│   └── types.py                          # cross-cutting dataclasses
├── tests/
│   ├── conftest.py
│   ├── fixtures/                         # phase 11 — saved listing HTML
│   ├── test_config.py
│   ├── test_store.py
│   ├── test_filters.py
│   ├── test_scoring.py
│   ├── test_dedup.py
│   ├── test_notify.py
│   ├── test_orchestrator.py
│   ├── test_cli.py
│   └── test_e2e_dry_run.py
└── data/                                 # gitignored, runtime-generated
    ├── watcher.db                        # SQLite
    ├── logs/watcher-YYYY-MM-DD.jsonl
    └── deadletter/                       # failed notifications
```

---

## Phase 0 — Scaffolding & dependencies

**Goal**: Create the package, declare dependencies, set up tooling.

**Subagent inputs**: PRD.md, this plan.

**Tasks**:
1. Create `pyproject.toml` with `[project]` metadata, Python ≥3.11, entry point `marketplace-watcher = "marketplace_watcher.cli:app"`.
2. Dependencies (runtime):
   - `pydantic>=2.6` (config schema)
   - `pyyaml`
   - `typer[all]`
   - `croniter`
   - `playwright` (post-install: `playwright install chromium`)
   - `playwright-stealth`
   - `httpx` (Discord webhook + retries)
   - `tenacity` (backoff)
   - `structlog`
   - `sentry-sdk`
   - `pgeocode`
   - `haversine`
   - `python-dateutil`
3. Dev dependencies: `pytest`, `pytest-asyncio`, `pytest-cov`, `ruff`, `mypy`, `respx` (HTTPX mocking), `freezegun`.
4. Create empty package skeleton (all dirs above with `__init__.py`).
5. Add `.gitignore` (data/, .venv/, __pycache__, .pytest_cache, *.db, .env).
6. Add `ruff` + `mypy` config to `pyproject.toml` (strict mypy on `src/`).
7. Create `src/marketplace_watcher/types.py` with shared dataclasses (see "Cross-cutting types" below).

**Cross-cutting types** (`types.py`):
```python
from dataclasses import dataclass, field
from datetime import datetime
from typing import Literal

@dataclass(frozen=True)
class RawListing:
    listing_id: str
    title: str
    description: str
    price: float | None
    currency: str
    location_text: str | None
    latitude: float | None
    longitude: float | None
    posted_at: datetime | None
    image_count: int
    first_image_url: str | None
    seller_display_name: str | None
    url: str

@dataclass(frozen=True)
class FilterResult:
    passed: bool
    failed_filter: str | None  # "max_price", "max_distance_miles", etc.
    detail: str | None

@dataclass(frozen=True)
class ScoreBreakdown:
    keyword_strength: float
    price_headroom: float
    distance: float
    freshness: float
    image_presence: float
    composite: float
    weights_used: dict[str, float]
    keyword_hits: list[dict]    # [{"term": str, "where": "title"|"description"}]
    threshold: float

@dataclass(frozen=True)
class ScoredItem:
    raw: RawListing
    filter_result: FilterResult
    score: ScoreBreakdown | None      # None if hard-filtered
    decision: Literal["dropped", "below_threshold", "would_notify", "notified", "duplicate"]

@dataclass(frozen=True)
class RunSummary:
    run_id: str
    search_id: str
    started_at: datetime
    finished_at: datetime
    examined: int
    hard_filtered: dict[str, int]   # reason → count
    scored: int
    notified: int
    skipped_incomplete: int
    duplicates: int
    errors: list[dict]
```

**Output**: Package installs cleanly with `pip install -e .[dev]`. `marketplace-watcher --help` runs (even if it just prints "no commands yet"). `pytest` runs (no tests yet, exits 0).

---

## Phase 1 — Storage layer

**Goal**: SQLite schema, connection management, migration runner, item + run repositories.

**Subagent inputs**: Phase 0 output, PRD §6.5 (dedup), §6.10 (logging), §7 (idempotency).

**Tasks**:

1. `store/db.py` — `connect(path: Path) -> sqlite3.Connection` opens SQLite with `PRAGMA journal_mode=WAL`, `foreign_keys=ON`, `synchronous=NORMAL`. Returns `sqlite3.Connection` with `row_factory = sqlite3.Row`.

2. `store/migrations.py` — `apply_migrations(conn)` runs the schema below idempotently using a `schema_migrations(version, applied_at)` table.

3. **Schema** (single migration file `001_init.sql` embedded as Python string):

```sql
CREATE TABLE searches (
  id TEXT PRIMARY KEY,                    -- stable id from config
  config_hash TEXT NOT NULL,              -- hash of search definition; for change detection
  paused INTEGER NOT NULL DEFAULT 0,
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL
);

CREATE TABLE runs (
  id TEXT PRIMARY KEY,                    -- uuid4
  search_id TEXT NOT NULL REFERENCES searches(id),
  started_at TEXT NOT NULL,
  finished_at TEXT,
  status TEXT NOT NULL,                   -- "running" | "ok" | "error" | "empty" | "all_filtered"
  examined INTEGER NOT NULL DEFAULT 0,
  hard_filtered_json TEXT,                -- JSON: reason -> count
  scored INTEGER NOT NULL DEFAULT 0,
  notified INTEGER NOT NULL DEFAULT 0,
  skipped_incomplete INTEGER NOT NULL DEFAULT 0,
  duplicates INTEGER NOT NULL DEFAULT 0,
  error_count INTEGER NOT NULL DEFAULT 0,
  correlation_id TEXT NOT NULL,
  is_dry_run INTEGER NOT NULL DEFAULT 0
);
CREATE INDEX runs_search_started ON runs(search_id, started_at DESC);

CREATE TABLE examined_items (
  listing_id TEXT NOT NULL,
  search_id TEXT NOT NULL,
  run_id TEXT NOT NULL REFERENCES runs(id),
  examined_at TEXT NOT NULL,
  raw_json TEXT NOT NULL,                 -- full RawListing snapshot
  filter_result_json TEXT NOT NULL,
  score_json TEXT,                        -- ScoreBreakdown if scored
  decision TEXT NOT NULL,
  notified_at TEXT,
  notification_channel TEXT,
  PRIMARY KEY (listing_id, search_id)
);
CREATE INDEX examined_run ON examined_items(run_id);
CREATE INDEX examined_age ON examined_items(examined_at);

CREATE TABLE search_health (
  search_id TEXT PRIMARY KEY REFERENCES searches(id),
  last_run_at TEXT,
  last_success_at TEXT,
  last_failure_at TEXT,
  consecutive_failures INTEGER NOT NULL DEFAULT 0,
  last_examined INTEGER,                  -- examined count from last run
  rolling_examined_mean REAL,             -- 30-day rolling
  last_notification_at TEXT
);

CREATE TABLE deadletter_notifications (
  id TEXT PRIMARY KEY,                    -- uuid4
  listing_id TEXT NOT NULL,
  search_id TEXT NOT NULL,
  payload_json TEXT NOT NULL,
  channel TEXT NOT NULL,
  failed_at TEXT NOT NULL,
  attempts INTEGER NOT NULL,
  last_error TEXT NOT NULL
);

CREATE TABLE schema_migrations (
  version INTEGER PRIMARY KEY,
  applied_at TEXT NOT NULL
);
```

4. `store/items.py`:
   - `was_seen(conn, listing_id: str, search_id: str) -> bool`
   - `record_examined(conn, *, listing: RawListing, search_id: str, run_id: str, filter_result, score, decision, notified_at=None) -> None` — wraps in `INSERT OR IGNORE` so re-runs are safe.
   - `mark_notified(conn, listing_id, search_id, channel, ts) -> None`
   - `prune_older_than(conn, cutoff: datetime) -> int` — returns rows deleted.
   - `get_item(conn, listing_id, search_id) -> dict | None` — for `inspect-item` CLI.

5. `store/runs.py`:
   - `start_run(conn, *, search_id, correlation_id, is_dry_run) -> str` (returns run_id, status="running").
   - `finish_run(conn, run_id, summary: RunSummary, status: str) -> None`
   - `get_recent_runs(conn, search_id, limit=20) -> list[dict]`
   - `prune_older_than(conn, cutoff: datetime) -> int`

6. **Atomic-run guarantee**: `start_run` + the body's `record_examined` calls happen inside a single transaction wrapper that commits at the end of the search run, OR commits per-item if the run is long. **Decision: commit per-item.** Reason: if a Marketplace block kills the browser mid-run, items already examined and notified must remain "seen"; otherwise re-runs cause duplicate notifications. Idempotency comes from `INSERT OR IGNORE` on `examined_items.PRIMARY KEY (listing_id, search_id)` and from "notification dispatch happens after `record_examined` commit succeeds" — see Phase 7.

**Output**: `pytest tests/test_store.py` passes:
- migrations idempotent (run twice, no error)
- `was_seen` + `record_examined` round-trip
- `prune_older_than` deletes only matching rows
- WAL file exists after first write

---

## Phase 2 — Search config schema & loader

**Goal**: YAML-driven config with strict validation.

**Subagent inputs**: Phase 0, PRD §6.1.

**Config file location resolution** (in order):
1. `--config PATH` CLI flag
2. `MARKETPLACE_WATCHER_CONFIG` env var
3. `$XDG_CONFIG_HOME/marketplace-watcher/config.yaml`
4. `~/.config/marketplace-watcher/config.yaml`

**Tasks**:

1. `config/schema.py` — Pydantic v2 models:

```python
class GlobalScoreWeights(BaseModel):
    keyword_strength: float = 0.35
    price_headroom: float = 0.25
    distance: float = 0.20
    freshness: float = 0.12
    image_presence: float = 0.08

    @model_validator(mode="after")
    def sum_to_one(self):
        s = sum(self.model_dump().values())
        if not (0.999 <= s <= 1.001):
            raise ValueError(f"weights must sum to 1.0, got {s}")
        return self

class SearchConfig(BaseModel):
    id: str = Field(pattern=r"^[a-z0-9_-]+$")
    name: str
    query: str
    origin_zip: str
    max_distance_miles: float = Field(gt=0)
    max_price: float = Field(gt=0)
    min_price: float | None = Field(default=None, ge=0)
    result_cap_per_run: int = Field(default=50, ge=1, le=500)
    keywords_must_contain_any: list[str] = []
    keywords_must_not_contain: list[str] = []
    keywords_boost: list[str] = []
    score_weights: dict[str, float] | None = None  # partial override
    threshold: float = Field(default=75, ge=0, le=100)
    schedule: str  # cron expression — validated via croniter
    paused: bool = False

class NotificationConfig(BaseModel):
    discord_webhook_url: SecretStr | None = None  # or read from DISCORD_WEBHOOK_URL env
    health_alert_channel: Literal["same"] = "same"  # placeholder for future routing

class StorageConfig(BaseModel):
    db_path: Path = Path("data/watcher.db")
    log_dir: Path = Path("data/logs")
    deadletter_dir: Path = Path("data/deadletter")

class RetentionConfig(BaseModel):
    items_days: int = 180   # 6 months
    runs_days: int = 365    # 12 months

class ObservabilityConfig(BaseModel):
    sentry_dsn: SecretStr | None = None  # or env SENTRY_DSN
    log_level: Literal["debug", "info", "warning", "error"] = "info"

class WatcherConfig(BaseModel):
    notifications: NotificationConfig = NotificationConfig()
    storage: StorageConfig = StorageConfig()
    retention: RetentionConfig = RetentionConfig()
    observability: ObservabilityConfig = ObservabilityConfig()
    global_score_weights: GlobalScoreWeights = GlobalScoreWeights()
    searches: list[SearchConfig]

    @model_validator(mode="after")
    def unique_ids(self):
        ids = [s.id for s in self.searches]
        if len(ids) != len(set(ids)):
            raise ValueError("search ids must be unique")
        return self
```

2. `config/loader.py`:
   - `load_config(path: Path | None = None) -> WatcherConfig` — resolves path, reads YAML, validates via pydantic, returns instance.
   - Cron validation: each `schedule` is run through `croniter.croniter(expr, base=datetime.now())` to ensure parseable.
   - Secrets: if `discord_webhook_url` is missing in YAML, fall back to `DISCORD_WEBHOOK_URL` env. Same pattern for Sentry.
   - `effective_weights(search: SearchConfig, global_w: GlobalScoreWeights) -> dict[str, float]` — applies partial override to global, normalizes to sum=1.0 (any zero override means that component is removed from the average).
   - `compute_search_hash(search: SearchConfig) -> str` — sha256 of canonical JSON representation. Used by store to detect config changes (logged with diff).

**Output**: `pytest tests/test_config.py` passes — invalid YAML raises with helpful messages, env-var fallback works, weight normalization is correct, duplicate IDs rejected.

---

## Phase 3 — Marketplace fetcher (Playwright)

**Goal**: Given a `SearchConfig`, return up to N `RawListing` objects from Facebook Marketplace.

**Subagent inputs**: Phases 0–2, PRD §6.3, §6.11.

**This is the riskiest phase.** Marketplace breaks scrapers periodically. The fetcher must be resilient and fail loudly with diagnostic context.

**Tasks**:

1. `fetch/base.py`:
```python
class Fetcher(Protocol):
    async def fetch(self, search: SearchConfig) -> list[RawListing]: ...
```

2. `fetch/marketplace.py` — `MarketplaceFetcher`:
   - Uses `playwright.async_api`.
   - Reuses a single browser context across calls within one process invocation. Lazy-init.
   - Stealth: `playwright_stealth.stealth_async(page)` on each new page.
   - Persistent context directory: `data/.browser-state/` so cookies/login persist across runs. **Login is manual**: first-time user runs `marketplace-watcher login` (Phase 9) which opens non-headless Chromium for them to log in once.
   - Search URL: `https://www.facebook.com/marketplace/search/?query={urlencoded(query)}&minPrice={min}&maxPrice={max}&radius={miles}&exact=false`. Origin zip is set via the location modal — use a session-stored location set during `login`.
   - Wait for `[role="main"] a[href*="/marketplace/item/"]` to appear (timeout 20s).
   - Scroll until `result_cap_per_run` items are rendered or 5 consecutive scrolls add nothing.
   - For each listing tile: extract listing URL → open in new tab → use `extract.parse_listing_page(html)` → close tab.
   - **Throttling**: 2–4s jittered delay between listing-detail fetches. Random user-agent rotation NOT done (stable UA looks more legitimate). Honor `result_cap_per_run` strictly.

3. `fetch/extract.py` — pure function `parse_listing_page(html: str, url: str) -> RawListing | None`:
   - Marketplace embeds listing data as JSON inside `<script>` tags (look for `"marketplace_listing_renderable_target"`). Parse it.
   - Fallback: structured-data `<meta property="og:*">` tags.
   - Extract: id (from URL or JSON), title, description, price (numeric, currency), location (city/state text + lat/lon if present), posted timestamp (Unix seconds → UTC datetime), image count (number of `og:image` or JSON image array length), first image URL, seller display name.
   - Returns `None` if any required field (title, price, listing_id, url) is missing — orchestrator will record as "skipped — incomplete data" with the missing-field name.

4. **Failure modes**:
   - **Login expired**: detect via redirect to `/login` or "Log in to continue" text → raise `MarketplaceAuthError`. Orchestrator surfaces as system alert.
   - **Page-shape change**: `parse_listing_page` returns None for >50% of items in a run → raise `MarketplacePageShapeError` after run. Orchestrator surfaces as system alert.
   - **CAPTCHA / block**: detect via known DOM signatures (`form[action*="checkpoint"]`, "We've temporarily restricted") → raise `MarketplaceBlockedError`. Orchestrator aborts the run.
   - **Network**: standard Playwright timeouts → raise; tenacity retries 3× with exponential backoff at the *page-level only*, not the run level.

5. **Test fetcher**: `fetch/marketplace.py` also exposes `FixtureFetcher(directory: Path)` that reads pre-saved JSON files matching `RawListing` shape. Used by tests and by `--fixture-dir` CLI flag for E2E without hitting Marketplace.

**Output**: `pytest tests/test_extract.py` passes against committed fixture HTML files in `tests/fixtures/`. A manual smoke test (`marketplace-watcher fetch --search SEARCH_ID --no-store --no-notify`, added in Phase 9) returns ≥1 listing for a real search after `marketplace-watcher login`.

---

## Phase 4 — Hard filters & scoring engine (parallelizable with Phase 3)

**Goal**: Pure deterministic scoring of a `RawListing` against a `SearchConfig`.

**Subagent inputs**: Phases 0–2, PRD §6.4.

**This is fully testable without I/O.** No network, no DB. Critical that this phase has dense unit-test coverage.

**Tasks**:

1. `score/geo.py`:
   - `class GeoLookup` — wraps `pgeocode.Nominatim('us')`. Caches `zip → (lat, lon)` in memory.
   - `distance_miles(zip_a, zip_b) -> float | None` — haversine.
   - `distance_miles_from_zip(zip_, lat, lon) -> float | None` — when listing has explicit lat/lon.

2. `score/filters.py` — `apply_hard_filters(listing, search) -> FilterResult`. Order:
   1. Missing required field (price, title) → `failed_filter="incomplete_data"`
   2. Missing location (no lat/lon AND no derivable distance) → `failed_filter="missing_location"`
   3. `price < min_price` or `price > max_price` → `failed_filter="price_out_of_range"`
   4. `distance > max_distance_miles` → `failed_filter="distance"`
   5. Any `keywords_must_not_contain` term in title or description (case-insensitive, regex with `\b` word boundaries) → `failed_filter="excluded_keyword"`, detail = which term
   6. `keywords_must_contain_any` non-empty AND no term hits → `failed_filter="missing_required_keyword"`

3. `score/components.py` — five pure functions, each returning `float ∈ [0, 100]`:

   - **`keyword_strength(title, description, must_contain, boost) -> tuple[float, list[hit]]`**
     - Title hits worth 2× description hits.
     - All keywords (must + boost) treated identically once must-contain filter is satisfied.
     - Per-keyword weight: 1.0 for description hit, 2.0 for title hit, multiple hits per keyword cap at the title-hit value (no double-counting same keyword in same field).
     - Density component: `min(1.0, total_hit_weight / total_keywords)` × 60.
     - Multi-keyword bonus (nonlinear): `pow(unique_keywords_hit / total_keywords, 0.6) × 40`. So 1/3 keywords hit = 33% × 40 ≈ 21pts; 3/3 = 100% × 40 = 40pts. Saturates near full coverage.
     - Returns `(score, hits_list)` where each hit = `{"term": "...", "where": "title"|"description"}`.
     - Edge case: zero total keywords → return `(50.0, [])` (neutral; can't measure).

   - **`price_headroom(price, max_price, min_price=None) -> float`**
     - `(max_price - price) / max_price × 100`, clamped to `[0, 100]`.
     - If `min_price` set: when `price == min_price` → 100, when `price == max_price` → 0, linear in between.

   - **`distance_score(miles, max_miles) -> float`**
     - `max(0, (max_miles - miles) / max_miles) × 100`. Floors at 0 once `miles ≥ max_miles`.

   - **`freshness(posted_at, now) -> float`**
     - Age in days. `0–1 days → 100`, `30+ days → 0`, linear in between (`100 × (1 - age_days/30)`).

   - **`image_presence(image_count) -> float`**
     - `0 → 0`, `1 → 50`, `2 → 80`, `3+ → 100`.

4. `score/engine.py` — `score(listing, search, global_weights, geo) -> ScoreBreakdown`:
   - Resolve effective weights (per-search override, then normalize so sum=1.0).
   - Compute each component.
   - Composite = weighted average of components.
   - Build `ScoreBreakdown` with all sub-scores, hits, weights-used, threshold.

5. **Determinism guarantee**: no `datetime.now()` inside score — `now` is injected. No randomness anywhere. No DB or network access. Same inputs → same outputs.

**Output**: `pytest tests/test_filters.py` and `tests/test_scoring.py` pass with ≥40 cases covering boundaries:
- Empty must-contain list → all listings pass keyword filter
- Zero keywords → keyword_strength = 50
- price = max_price → price_headroom = 0
- miles = max_miles → distance = 0
- age = 30 days → freshness = 0; age = 31 days → freshness = 0 (floored)
- image_count = 0 → image_presence = 0
- Effective weights with one component zeroed sum to 1.0 across the rest
- Same input twice produces byte-identical breakdown

---

## Phase 5 — Dedup orchestration glue

**Goal**: Thin layer combining the store with the scorer. Mostly already in Phase 1; this phase adds the higher-level "evaluate one listing end-to-end" function used by the orchestrator.

**Subagent inputs**: Phases 1, 4.

**Tasks**:

1. `score/engine.py` (extend) — `evaluate(listing, search, global_weights, geo, conn, run_id, dry_run) -> ScoredItem`:
   - Check `was_seen(conn, listing.listing_id, search.id)` → if seen, return `ScoredItem(decision="duplicate")`.
   - Apply hard filters → if dropped, build `ScoredItem(decision="dropped")`. Persist with `record_examined` (unless `dry_run`).
   - Score → if composite < threshold, decision = `"below_threshold"`. Persist (unless `dry_run`).
   - Else decision = `"would_notify"`. Persist (unless `dry_run`). Notification dispatch happens later in orchestrator (Phase 7).

2. **Dry-run behavior**: when `dry_run=True`:
   - Dedup check: still run (so dry-run output reflects what *would* happen).
   - Persist: `False`. Don't write to `examined_items`. Don't update `runs.examined` counters in DB (track in-memory).
   - Notification: never dispatched.

**Output**: `pytest tests/test_dedup.py` passes — second run over same fixture set produces zero new "would_notify" decisions and all items show `decision="duplicate"`.

---

## Phase 6 — Notification dispatcher

**Goal**: Channel-agnostic dispatcher with Discord webhook implementation.

**Subagent inputs**: Phases 0–2, PRD §6.6, §6.11.

**Tasks**:

1. `notify/payload.py` — `NotifiableItem` dataclass (the *interface*, channel-agnostic):
```python
@dataclass(frozen=True)
class NotifiableItem:
    listing_id: str
    search_id: str
    search_name: str
    title: str
    price: float
    currency: str
    distance_miles: float
    composite_score: float
    threshold: float
    matched_keywords: list[dict]   # top 1–3 from breakdown.keyword_hits
    posted_at: datetime | None
    first_image_url: str | None
    listing_url: str
    message_kind: Literal["match", "system_alert"] = "match"
    alert_text: str | None = None  # populated only for system_alert
```

2. `notify/base.py`:
```python
class NotificationDispatcher(Protocol):
    async def dispatch(self, item: NotifiableItem) -> DispatchResult: ...

@dataclass(frozen=True)
class DispatchResult:
    success: bool
    channel: str
    attempts: int
    error: str | None
    sent_at: datetime | None
```

3. `notify/discord.py` — `DiscordWebhookDispatcher`:
   - Sends an embed via webhook POST.
   - **Match embed**:
     - Color: green (`0x57F287`)
     - Title: `${title}` (truncate 256 chars)
     - URL: listing URL
     - Description: `**$${price}** · ${distance:.0f} mi · posted ${posted_ago_human}`
     - Footer: `${search_name} · score ${score:.0f}/${threshold:.0f}`
     - Fields: "Matched keywords" → `term1 (title), term2 (description)`
     - Thumbnail: `first_image_url`
   - **System-alert embed**:
     - Color: red (`0xED4245`)
     - Title: `⚠ ${search_name}: ${alert_text}` (no listing URL)
   - Discord webhook rate limits (5 req/2s) handled by `httpx` + `tenacity`: 3 attempts with exponential backoff (2, 4, 8s, +jitter), respecting `Retry-After` header.

4. `notify/retry.py` — `with_retry(coro, attempts=5, base=2.0) -> DispatchResult`. After exhaustion: writes JSON to `data/deadletter/${listing_id}-${search_id}-${ts}.json` AND inserts row into `deadletter_notifications`.

5. **Pluggability**: `NotificationDispatcher` is a Protocol; the orchestrator's constructor takes a `dispatcher: NotificationDispatcher`. Future channels (interactive-triage, per-search routing, email) implement the same protocol and replace `DiscordWebhookDispatcher` without changes elsewhere.

**Output**: `pytest tests/test_notify.py` (using `respx` to mock Discord webhook):
- Successful send → `DispatchResult.success=True`, attempts=1
- 3× 500 → exhaustion → deadletter file written, deadletter row inserted, returns `success=False`
- 429 with `Retry-After: 5` → respected
- Embed JSON shape matches Discord API spec (validate against schema)

---

## Phase 7 — Run orchestrator & scheduler

**Goal**: The "single entrypoint" the external cron calls — figures out which searches are due, runs them sequentially, persists results, dispatches notifications.

**Subagent inputs**: Phases 1–6, PRD §6.2, §7 (idempotency).

**Tasks**:

1. `orchestrator/scheduler.py`:
   - `due_searches(searches: list[SearchConfig], runs_repo, now: datetime) -> list[SearchConfig]`:
     - For each non-paused search, look up last run's `started_at`.
     - Compute `croniter(search.schedule, last_started).get_next(datetime)`. If `<= now`, search is due.
     - First-ever run for a search: due immediately.
   - Order: stable by `search.id` for reproducibility.

2. `orchestrator/run.py` — `async run_search(search, *, ctx) -> RunSummary` where `ctx` carries DB conn, fetcher, dispatcher, geo, global_weights, logger, dry_run flag:
   - `run_id = start_run(...)`. correlation_id = run_id.
   - `try`: `listings = await fetcher.fetch(search)`.
   - For each listing:
     - `correlation_id_per_item = f"{run_id}:{listing.listing_id}"`
     - `evaluate(...)` (Phase 5).
     - If `decision == "would_notify"` and not dry-run: build `NotifiableItem`, `await dispatcher.dispatch(...)`. On success → `mark_notified` and update `decision="notified"` in stored row. On failure → leave row at `would_notify`, deadletter row already written, increment `error_count`.
     - Log structured event for every step (filtered, scored, notified, etc.) with `run_id`, `search_id`, `correlation_id`, listing fields.
   - `finally`: `finish_run(run_id, summary, status)`.
   - **Status determination**:
     - `error` if exceptions (auth/blocked/page-shape) raised
     - `empty` if `examined == 0`
     - `all_filtered` if `examined > 0 && scored == 0`
     - `ok` otherwise
   - **System-alert dispatch**: on `MarketplaceAuthError`, `MarketplaceBlockedError`, `MarketplacePageShapeError` → build `NotifiableItem(message_kind="system_alert", alert_text=...)` and dispatch.
   - **Health update**: increment `consecutive_failures` on error status; reset to 0 on `ok`/`empty`. Update `last_run_at`, `last_success_at`, `last_failure_at`, rolling-mean fields.

3. `orchestrator/run.py` — `async run_due(config: WatcherConfig, *, dry_run=False, only_search_id: str | None = None) -> list[RunSummary]`:
   - Bootstraps DB connection, geo, fetcher, dispatcher.
   - Resolves due searches (or just `only_search_id` if given).
   - **Sequential** — per PRD §6.2 v1 constraint.
   - Returns list of summaries for CLI reporting.

4. **Idempotency**:
   - Within a run, `INSERT OR IGNORE` on the `(listing_id, search_id)` PK ensures re-running is safe.
   - Notification commits: order is `(persist examined row with decision="would_notify") → (dispatch) → (update row to "notified")`. If process crashes between persist and dispatch, the listing is recorded as seen → it will NOT be retried → the user does not get duplicate notifications. The trade-off (PRD §7 "no duplicate notifications" wins over "no missed notifications mid-crash") is the documented choice.

**Output**: `pytest tests/test_orchestrator.py` passes with `FixtureFetcher` + mocked dispatcher:
- Two consecutive `run_due` calls produce zero notifications on the second pass.
- Mid-run dispatcher exception leaves item at `decision="would_notify"`, `error_count > 0`.
- `dry_run=True` produces zero DB writes and zero dispatcher calls but returns a populated `RunSummary`.

---

## Phase 8 — Logging & observability

**Goal**: Structured logs to local JSONL files + Sentry error reporting. Per-run summary log line.

**Subagent inputs**: Phases 0, 7.

**Tasks**:

1. `obs/logging.py`:
   - `configure(log_dir: Path, level: str)` — structlog with JSON renderer. File handler rotates daily by date (`watcher-YYYY-MM-DD.jsonl`). Stderr handler for level ≥ warning.
   - Bind `run_id`, `search_id`, `correlation_id` via `structlog.contextvars`.
   - Standard fields on every event: `ts`, `level`, `event`, `run_id`, `search_id`, `correlation_id`, `module`.

2. `obs/errors.py`:
   - `init_sentry(dsn: SecretStr | None)` — no-op if dsn is None.
   - `capture_exception(exc, context: dict)` — wraps Sentry SDK with the correlation_id baked into tags so AI agents can later query Sentry by run_id/search_id/correlation_id.

3. **Run-summary log line**: `orchestrator/run.py` emits one event `event="run_summary"` at finish with all `RunSummary` fields. This is the "did anything weird happen overnight?" surface.

4. **Anomaly hooks**: Phase 10 reads these structured logs (or the DB, equivalently) to detect anomalies. No work in this phase beyond making sure the structured fields exist.

**Output**: `pytest tests/test_logging.py` passes — log lines are valid JSON, contain all required fields, and Sentry init is a no-op when DSN is missing.

---

## Phase 9 — CLI

**Goal**: Scriptable surface for v1 and for future agent integration.

**Subagent inputs**: Phases 1–8.

**Commands** (`marketplace-watcher <cmd>`):

| Command | Purpose |
|---|---|
| `run [--config PATH] [--only SEARCH_ID]` | Run all due searches; entrypoint for cron. Exit 0 on all-success, 1 on any error. |
| `dry-run --search SEARCH_ID [--config PATH]` | PRD §6.9. Runs end-to-end, no persistence, no notifications. Reports counts + score breakdown for would-notify items as JSON. Use `--format=json` (default) or `--format=pretty`. |
| `list-searches` | JSON dump of search configs + last-run summary + paused state. |
| `pause SEARCH_ID` / `resume SEARCH_ID` | Toggles `searches.paused` in DB and updates config? **Decision: pause/resume modify the DB only**, not the YAML — YAML is the declarative source of truth and pause is operational state. CLI-set paused state is overlaid on YAML at load time. |
| `add-search --file PATH` | Reads a single SearchConfig YAML file and merges it into the main config. Validates first, fails atomically. Future agent hook. |
| `remove-search SEARCH_ID` | Removes from config (with confirm prompt unless `--yes`). |
| `inspect-item LISTING_ID --search SEARCH_ID` | Returns the full `examined_items` row + score breakdown as JSON. PRD §U6 explainability. |
| `prune` | Manually triggers retention pruning. `run` also triggers it daily (per first run of the day). |
| `login` | Opens non-headless Chromium for one-time Marketplace login + location set. Persists state to `data/.browser-state/`. |
| `health` | Prints per-search health snapshot (last run, consecutive failures, anomalies) as JSON. |
| `version` | Prints package version. |

**JSON-first output**: every command supports `--format=json` (default for non-TTY) and `--format=pretty` (default for TTY). All structured commands output dicts that are agent-consumable. No human-only formatting in the JSON path.

**Tasks**:

1. `cli.py` — typer app with subcommands. Each command thin: parses args, calls into `orchestrator/run.py` or store repos.
2. Exit codes: 0 = success, 1 = operational error, 2 = config error, 3 = no due searches (informational, not failure).
3. `--config` is a global option.

**Output**: `pytest tests/test_cli.py` passes — invokes typer's CLI runner against each command with mocked DB/fetcher/dispatcher.

---

## Phase 10 — Health & anomaly surface (parallelizable with Phase 11)

**Goal**: PRD §6.10 health and anomaly surface; PRD §6.11 distinct error events.

**Subagent inputs**: Phases 1, 7, 8.

**Tasks**:

1. `obs/health.py`:
   - `compute_health(conn) -> list[HealthStatus]` per search: last_run_at, last_success_at, consecutive_failures, last_examined, rolling_examined_mean (30-day), last_notification_at.
   - `detect_anomalies(conn) -> list[Anomaly]`:
     - **Empty drift**: search with rolling_examined_mean ≥ 5 returned 0 examined for 2 consecutive runs.
     - **All-filtered drift**: search with `scored == 0 && examined > 0` for 3 consecutive runs.
     - **Failure streak**: `consecutive_failures >= 3`.
     - **Latency drift** (deferred): just emit metric; don't alert in v1.
   - Anomaly objects → optional Discord system-alert dispatch (configurable, on by default). Dispatch is rate-limited so the user isn't spammed: max one alert per anomaly type per search per 24h, tracked via in-memory or DB key.

2. `cli health` command shows health + anomalies.

3. **Hook into `run`**: at the end of `run_due`, call `detect_anomalies` and dispatch any new alerts.

**Output**: `pytest tests/test_health.py` covers the four anomaly conditions with synthetic run history.

---

## Phase 11 — Tests (parallelizable with Phase 10)

**Goal**: Comprehensive test suite. Most tests already specified per phase; this phase ensures coverage and adds the E2E test.

**Subagent inputs**: All prior phases.

**Tasks**:

1. `tests/conftest.py`:
   - `tmp_db` fixture (in-memory SQLite with migrations applied).
   - `frozen_now` fixture using `freezegun`.
   - `geo` fixture stubbed with `pgeocode` over a tiny zip set.
   - `fixture_fetcher` returning canned `RawListing` objects.
   - `mock_dispatcher` recording calls, returning configurable success/failure.

2. `tests/fixtures/listings/` — 10–20 hand-curated `RawListing` JSON files representing realistic scenarios: cheap-bargain, far-away, has-excluded-keyword, no-images, all-keywords-hit, missing-price, etc.

3. `tests/fixtures/marketplace_html/` — 3 saved Marketplace listing detail pages for `extract.py` testing.

4. **E2E dry-run test** (`tests/test_e2e_dry_run.py`):
   - Loads a synthetic config with one search.
   - Uses `FixtureFetcher`.
   - Runs `run_due(dry_run=True)`.
   - Asserts: zero DB rows in `examined_items`, zero dispatcher calls, returned `RunSummary` reports correct counts.

5. **E2E full-run test** (`tests/test_e2e_full_run.py`):
   - Same as above, dry_run=False, mock dispatcher.
   - First run: N notifications dispatched, N rows in `examined_items`.
   - Second run (same fixtures): 0 notifications, all rows show `decision="duplicate"` for newly examined ones (or no new examined since fixtures are deduped).

6. Coverage target: ≥85% on `score/`, `store/`, `orchestrator/`, `notify/`, `config/`. Lower acceptable on `fetch/marketplace.py` (Playwright integration is exercised manually).

**Output**: `pytest --cov` passes, coverage gates met.

---

## Phase 12 — Docker + cron host (parallelizable with Phase 13)

**Goal**: Make this trivially deployable on the user's existing Docker host.

**Subagent inputs**: All prior phases, user's `CLAUDE.md` (Docker workflows, no sudo).

**Tasks**:

1. `Dockerfile`:
   - `FROM python:3.11-slim`
   - Install Playwright system deps + Chromium (`playwright install --with-deps chromium`).
   - Copy package, `pip install -e .`.
   - Default CMD: `marketplace-watcher run`.
   - Volumes: `/data` (persistent SQLite, logs, browser state).

2. `docker-compose.yml`:
   - Service `marketplace-watcher`.
   - Mounts `./data:/data` and `./config.yaml:/etc/marketplace-watcher/config.yaml:ro`.
   - Env: `DISCORD_WEBHOOK_URL`, `SENTRY_DSN`, `MARKETPLACE_WATCHER_CONFIG=/etc/marketplace-watcher/config.yaml`.
   - Restart: `unless-stopped`. **But** the container shouldn't run as a daemon — `marketplace-watcher run` is one-shot.
   - **Decision**: use `restart: "no"` and trigger via host cron OR sidecar `ofelia` job scheduler in compose. Document both; default is host cron with `docker compose run --rm marketplace-watcher run`.

3. `config.example.yaml` — fully annotated example showing every option, two example searches.

4. README of three sections:
   - Setup (clone, install, `playwright install chromium`, `marketplace-watcher login`, edit config)
   - Cron setup (sample crontab line)
   - Operations (dry-run, inspect-item, pause/resume, health)

**Output**: `docker compose run --rm marketplace-watcher dry-run --search example` works against `config.example.yaml` (with FixtureFetcher swap).

---

## Phase 13 — Documentation & finishing touches

**Goal**: Onboarding doc, examples, and the small things that make the project usable a year from now.

**Subagent inputs**: All phases.

**Tasks**:

1. `README.md` — what it is, why, and how to run. Link prominently to `PRD.md`.
2. `config.example.yaml` (if not done in 12) — full annotated example.
3. `docs/scoring.md` — explain the formula in words so the user can tune weights without re-reading the code.
4. `docs/operations.md` — run-book: how to investigate a missed notification (use `inspect-item` and structured logs), how to handle Marketplace blocking, how to rotate webhook.
5. `CHANGELOG.md` seeded with v0.1.
6. Top-of-`__main__.py` and module docstrings on every package.
7. `mypy --strict` clean on `src/`. `ruff check` clean.

**Output**: A user (or Sonnet, in a future session) can start the project from a fresh checkout following only the README.

---

## Verification (end-to-end)

After Sonnet finishes all 13 phases:

1. `pip install -e .[dev]` succeeds.
2. `playwright install chromium` succeeds.
3. `pytest --cov` passes; coverage targets met.
4. `mypy --strict src/` clean.
5. `ruff check` clean.
6. `marketplace-watcher --help` lists all commands.
7. With `config.example.yaml` (and a real Discord webhook), `marketplace-watcher dry-run --search example` (using FixtureFetcher via env override) returns a valid JSON report.
8. `marketplace-watcher login` opens Chromium successfully.
9. After login + a real config, `marketplace-watcher run --only SEARCH_ID` produces a Discord notification end-to-end. (Manual gate; not automated.)
10. Re-running step 9 produces zero new notifications.
11. `marketplace-watcher inspect-item LISTING_ID --search SEARCH_ID` returns the full breakdown.
12. Killing the process mid-run and restarting does not produce duplicate notifications.

---

## Decisions made under ambiguity (and reasoning)

The PRD left several questions open or implied. My calls:

1. **Language: Python.** Best library ecosystem for scraping + Discord + structured logging + zip-distance. Aligns with the user's existing stack (per `CLAUDE.md`). Single-binary alternatives (Go, Rust) buy nothing for a personal one-host automation.

2. **Marketplace fetch: Playwright headless Chromium with stealth, persistent context.** Direct HTTP scraping breaks weekly. Playwright with persistent login is the only approach with a chance of staying alive past initial deploy without weekly maintenance. The `marketplace-watcher login` command keeps the auth cost on the user, once, manually. Trade-off: heavier resource usage (browser per run) vs. fragility — accepted.

3. **Storage: SQLite, WAL mode, single file at `data/watcher.db`.** Single-user, low write volume, atomic, no daemon. Postgres would be massive overkill. WAL gives concurrent-read safety for the future "multi-search-running-in-parallel" extensibility hook (PRD §8) without changes.

4. **Config: YAML.** Aligns with user's stack. Pydantic v2 enforces schema. Search config is the source of truth; pause is operational state in DB (so `pause` doesn't mutate the user's YAML).

5. **Idempotency choice for "crash between persist and dispatch":** when the orchestrator persists an item as `decision="would_notify"` and then the process dies before dispatch, the item is recorded as seen and will NOT be retried on next run. Per PRD §4 "Do not over-notify" anti-goal and §7 "no duplicate notifications," missing one notification on rare crash is better than duplicating. Documented in run-book. Failed-after-persist items still go to deadletter table when the dispatcher itself fails (vs. process crash) so the user can hand-recover.

6. **Notification config: webhook URL only for v1.** Question §10.2. Webhook is enough for v1 and lower-friction. Bot token + channel ID is needed only for richer interactions (buttons, threads), which are PRD §8 out-of-scope. The dispatcher abstraction means switching later is a 1-class change.

7. **Run record retention: 12 months hard, no aggregate-stats forever.** Question §10.3. Personal-scale data, simple pruning, no need for time-series aggregation now. Adding it later is straightforward.

8. **Notification retry backoff: 5 attempts, base=2s, exponential, +jitter, honoring `Retry-After`.** Question §10.4. Standard pattern. Total worst-case ~62s. Beyond that → deadletter. Sane for personal use.

9. **Distance precision: straight-line from ZIP centroid via pgeocode + haversine.** Question §10.5. PRD §10 lists this as the v1 default. Driving distance would require a paid API (Google/Mapbox). Anti-goal: paying money for marginal correctness on a personal automation.

10. **Logging vendor: local JSONL + Sentry for errors in v1.** Question §10.1. Local JSONL satisfies the structured-logs requirement and is queryable by `jq` or any agent. Sentry has a generous free tier, structured-event API, and great error-context support — both human and agent friendly. Vendor lock is minimal because `obs/logging.py` is a thin wrapper. The "spike a hosted logs vendor" can come later without code rework.

11. **Cron: external host cron drives `marketplace-watcher run` once per minute (or less often).** The orchestrator computes due-ness internally from the `schedule` field on each search. This honors PRD §6.2 ("triggered by an external system cron") and means the user adds exactly one crontab line for the whole product.

12. **CLI exit code 3 for "no due searches."** Allows the cron line to be `marketplace-watcher run; true` or for the user to alert only on actual errors (exit 1/2). Common operational nuance.

13. **Effective-weights normalization rule.** PRD §6.4 says overrides are "normalized so the active weights still sum to 1.0" but doesn't define "active." Decision: a per-search override that sets a component to 0 *removes* that component from the weighted average; remaining components are renormalized to sum to 1.0. Setting an override to a positive number means "use this exact value, then renormalize the whole set."

14. **Keyword bonus formula.** PRD §6.4 says "scales nonlinearly so 'all keywords matched' is meaningfully better." Decision: `pow(unique_hit / total, 0.6) × 40`. Concave, saturating, gives 1/3 → ~21pts and 3/3 → 40pts (≈2× the 1/3 score for 3× the matches — meaningful difference without dominating).

15. **Pause via DB, not YAML.** PRD §6.7 says paused state is preserved across resume. Mutating the user's YAML on every pause feels invasive and conflicts with declarative config. DB-stored paused flag overlaid at config-load time is cleaner and machine-edited.

16. **No worktree / branch isolation.** This is a greenfield personal project; standard `main` branch development is correct. The user's `CLAUDE.md` doesn't mandate otherwise.

17. **No CI in v1.** Personal project; the tests run locally. Adding GitHub Actions is trivial later but not in scope per "vibe coded plan."

These decisions are all reversible with localized changes — no architectural lock-in. If the user disagrees with any, they can pivot in a future session.
