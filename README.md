# Xcerebro County Intelligence Framework

**Copyright © 2026 Xcerebro LLC. All rights reserved.**
Licensed under the proprietary Xcerebro LLC VIP license. See `LICENSE.md`. This Framework is not open source; access is limited to active Xcerebro LLC VIP members and approved licensees.

Reusable harness for building county-level distress-lead intelligence systems in any county.

This is not a county-specific build. It is a portable shell. County-specific data lives in `config/counties/<slug>.json` and `scrapers/<source>.py` — never in the universal pipeline.

---

## Current state — v5.6.0 (stable)

**What works today:**

- The **staged pipeline** is executable end to end: normalize → classify → match → aggregate → score → review → dashboard (`scaffold/pipeline/`).
- **Phase 0 county recon** is a formalized protocol with 7 mandatory gap-closing steps and 37 automated gate tests.
- **Contract schemas** for every record shape in the pipeline, schema-validated.
- **Synthetic test harness** so a build can be exercised before real county data enters it.

**What is not done yet — read this before planning around it:**

- **Production live-browser verification and auto-rollback are NOT implemented.** §20 defines the contract and `scaffold/ops/semantic_verify_template.py` is a documentation-grade template. `verify_live.py` and `watchdog.py` are stubs. Per-county responsibility until universal tooling lands.
- **The Build Eligibility Gate is operator judgment, not an algorithm.** Protocol 01 §01.16 says so explicitly — the stepwise gate algorithm is not yet formalized. The justification trail is what makes the verdict reviewable.
- **Scrapers are per-county.** The framework ships no working adapter for any real portal. Every county writes its own against the fingerprint recon produces.
- **`scaffold/tests/v5_4_0_pending/`** holds behavioral specs that are red by design and quarantined out of the default gate.

---

## How a build actually runs

**This is an operator-supervised, checkpointed build. It is not a one-command autonomous install, and treating it as one produces bad county builds.**

Earlier versions of this README advertised a "type one sentence and watch it go" flow. That framing was removed in v5.6.0 because it set the wrong expectation: the parts of a county build that most need operator judgment — deciding whether a source is really blocked, whether a lead type actually exists in that jurisdiction, whether a feed is current — are exactly the parts that cannot be delegated blind. The framework is heavily automated *within* each phase and deliberately stops *between* phases.

The real loop:

1. **Clone this framework into your county-build repo.** The framework lives inside the county repo; the county repo is what you commit to.
2. **Bootstrap the run folder:**
   ```
   python scaffold/bootstrap_county.py --county "<Name> County" --state "<State>" --slug <county_state> --phase phase0
   ```
   This creates `runs/<slug>/` with a launch file and operator notes. It creates nothing else.
3. **Open Claude Code and point it at the launch file:**
   ```
   Read MASTER_PROMPT.md and runs/<slug>/LAUNCH_<SLUG>.md. Run Phase 0 only.
   ```
4. **Phase 0 — county source recon.** Expect this to be the longest and most interactive phase. It runs 15 mandatory discovery queries, a 5-layer verification gate per source, a 29-type lead sweep, and 7 required gap-closing steps (§01.22–§01.32). It will ask you things. Answer them — recon quality determines everything downstream.
5. **Review the change manifest and the build verdict.** This is a real review, not a formality. Check that blocked sources were actually tested, that lead types were checked against the county's own vocabulary, and that "live" feeds are actually current.
6. **Authorize Build Mode explicitly.** Phase 0 ends at a hard gate. Implicit approval is not accepted.
7. **Build one thin vertical slice first** — one adapter, end to end, proven — before scaling to the rest of the sources.
8. **Review at each subsequent phase gate.** Every phase ends with a manifest and a stop.

### What you will be asked to approve

Claude Code asks before running shell commands and before fetching web content. During a build expect approval requests for: web search and web fetch against official portal domains, `scaffold/bootstrap_county.py`, `scaffold/ops/write_county_config.py` (the only sanctioned way to write a county config), and `scaffold/tests/run_all.py`. Approving these broadly within the county repo is safe and expected.

### Hard boundaries

1. **The county config is written by the writer, never by hand.** If `write_county_config.py` returns `JSON_INVALID` or `SCHEMA_INVALID`, exactly one structured repair is attempted, then the build stops with `CONFIG_WRITE_FAILED`. It does not silently proceed.
2. **The framework is universal; the county is configured.** MASTER_PROMPT §4.31 forbids county-specific data in universal pipeline code. The same `scaffold/pipeline/` runs for every county.
3. **No P0 distress source, no build.** If no daily-refresh distress source is unblocked, Phase 0 halts with a verdict rather than filling a dashboard with parcel data.
4. **Recon is metadata-only.** No record scraping, no account creation, no payment, no CAPTCHA solving, no robots.txt bypass during Phase 0.

---

## What this framework does

**The product is fresh county-level distress intelligence with daily refresh.** The county is the moat. Daily refresh is non-negotiable. Fresh distress signals are the core asset. Enrichment data supports county intelligence; it never replaces it.

Every county built on this framework inherits:

**Distress ingestion (the moat):**
- Daily ingestion of fresh county distress filings: clerk recordings, court dockets, sheriff sales, code enforcement, tax delinquency
- Source priority tiers (P0 daily-distress / P1 weekly-distress / P2 enrichment) — Phase 0 build halts if no P0 source is unblocked
- Lifecycle reasoning over fresh filings — chronology, status engine, suppression of resolved signals (releases, satisfactions, discharges, dismissals)
- Source heartbeat and cursor tracking so daily refreshes don't duplicate or miss records
- Telegram alerts for new high-stack leads, source failures, session expiry, regressions

**Normalization and scoring:**
- Universal document normalization layer translating raw recorder/court abbreviations and OCR-corrupted text into canonical document types before scoring
- Source-classified, scored, deal-path-classified leads (wholesale / flip / sub-to / seller-finance / partial-interest / messy-title / rental-acquisition / dispo-only / do-not-pursue)
- Title complexity as a dimension separate from motivation, gating which deal paths are operationally viable
- Strict evidence ledger attached to every field and every claim

**Enrichment (supporting role only):**
- Entity resolution for individuals, LLCs, trusts, estates, parcels, addresses, cases, and instruments
- Parcel master / appraisal district enrichment for assessed value, equity proxy, owner mailing
- GIS / USPS vacancy / utility shutoff feeds where available

**Infrastructure:**
- GitHub private repo + GitHub Pages dashboard hosting (revocable client access)
- Optional Supabase database storage for production scale
- Live-browser verification and auto-rollback: contract defined in v5.3.0 (§20 Semantic Verification Contract). Production implementation deferred to a future harness release. v5.3.0 ships the contract surface and a documentation-grade reference implementation template (scaffold/ops/semantic_verify_template.py); production verifier and watchdog infrastructure are per-county responsibility until universal production tooling lands.
- Synthetic test harness before real county data enters the system

## Who this is for

Real estate operators who build lead-generation systems for investor clients. The framework's clients are wholesalers, flippers, creative-finance investors, partial-interest specialists, and messy-title investors. **They will physically call the leads this system produces.** Every architectural decision serves that.

## Universal rule

Do not hardcode a county. Do not hardcode a state. Do not carry assumptions from a previous county build. Each county is discovered from its own config and `RECON.md`.

## What's in this framework

```
xcerebro-county-intel/
├── MASTER_PROMPT.md              # paste this into Claude Code to start a county build
├── MIGRATION.md                  # operator handoff — read this if you're using the framework
├── README.md                     # this file
│
├── knowledge_base/
│   ├── domain/                   # the WHAT — investor-side knowledge
│   │   ├── 00_client_business_model.md      # who the leads are for
│   │   ├── 01_lead_types.md                 # 14-pattern taxonomy
│   │   ├── 02_signals_and_sources.md        # lead / enrichment / negative-signal classification
│   │   ├── 03_scoring_and_stacking.md       # 0-100 scoring with reasons
│   │   ├── 04_deal_path_classifier.md       # routes to 9 deal paths
│   │   ├── 05_review_queue_rules.md         # quality gate
│   │   ├── 06_hallucination_controls.md     # anti-fabrication rules
│   │   ├── 07_fallback_metrics.md           # 12 quality thresholds
│   │   ├── 08_document_normalization.md     # raw recorder/court abbrev → canonical type
│   │   ├── 09_document_lifecycle.md         # chronology, status engine, suppression
│   │   ├── 10_title_complexity.md           # title complexity as separate dimension
│   │   └── canonical_doc_types.json         # machine-readable canonical type registry
│   │
│   ├── architecture/             # the CONTRACTS — data shape and integrity
│   │   ├── 08_evidence_ledger.md            # every claim needs evidence
│   │   ├── 09_output_schemas.md             # 10 strict record shapes
│   │   ├── 10_source_heartbeat_and_cursors.md  # source health and freshness
│   │   ├── 11_database_and_storage.md       # STATIC / SUPABASE / HYBRID
│   │   ├── 12_entity_resolution.md          # when records refer to the same entity
│   │   ├── 13_lead_origination_contract.md  # what events may originate a lead
│   │   ├── 16_source_of_record_matrix.md    # the authoritative source per field
│   │   ├── 17_debtor_party_rules.md         # debtor-party identification rules
│   │   ├── 18_signal_aggregation_contract.md   # combining signals across sources
│   │   ├── 19_aggregator_idempotency_rule.md   # re-runs never duplicate or drift
│   │   └── 20_semantic_verification_contract.md  # semantic verification gate
│   │
│   ├── engineering/              # the HOW — build-side knowledge
│   │   ├── 00_tooling_decision_tree.md      # which tool for which job
│   │   ├── 01_python_environment.md         # Python 3.12, pinned deps
│   │   ├── 02_scraping_libraries.md         # requests, Playwright, etc.
│   │   ├── 03_document_readers.md           # PDF, DOCX, XLSX, CSV, HTML
│   │   ├── 04_blocked_source_strategies.md  # reCAPTCHA, WAF, paywalls, login walls
│   │   ├── 05_verification_and_rollback.md  # live-browser gate + auto-rollback
│   │   └── 06_deployment.md                 # GitHub Pages, scheduled tasks
│   │
│   └── protocols/                # the WHEN — phase sequencing
│       ├── 01_county_recon.md               # Phase 0 county source recon
│       └── 02_build_mode_protocol.md        # Build Mode phase sequencing
│
├── config/counties/              # per-county config — only thing that varies
│   ├── _schema.md                            # human-readable schema doc
│   ├── _schema.json                          # JSON Schema (validates configs)
│   └── _template.json                        # empty config to copy for new counties
│
├── runs/<slug>/                  # per-county run folder (launch file, notes,
│                                 # recon artifacts, phase gates)
│
└── scaffold/
    ├── bootstrap_county.py       # creates runs/<slug>/ — the only bootstrap step
    ├── pipeline/                 # THE EXECUTABLE PIPELINE
    │   ├── normalize.py classify.py matcher.py aggregator.py
    │   ├── score.py review.py dashboard.py run_pipeline_staged.py
    │   ├── contracts/            # JSON Schema record shapes
    │   └── translators/          # source -> canonical translators
    ├── ops/                      # write_county_config.py, PII guard,
    │                             # verify_live.py + watchdog.py (STUBS)
    ├── data/                     # synthetic test harness
    │   ├── synthetic_parcels.jsonl           # 12 parcels covering all scenarios
    │   ├── synthetic_signals.jsonl           # 24 signals across all 11 patterns
    │   └── synthetic_expectations.json       # what the build should produce
    └── tests/                    # 37 gate tests — run_all.py
        ├── v5_3_0/               # architecture-contract invariants
        ├── v5_4_0/               # pipeline contract-shape tests
        ├── v5_6_0/               # recon protocol Gap 4-7 invariants
        └── v5_4_0_pending/       # red by design, NOT in the default gate
```

## How to use it (detailed)

**For first-time use, see "How a build actually runs" at the top of this README, or read `START_HERE.md`.** The flow below expands the same path with the per-phase detail.

1. Read `MIGRATION.md` end-to-end.
2. Create a private GitHub repo for the county build (e.g. `<county-slug>-intel`).
3. Copy this directory into the new repo.
4. **Either** run `python scaffold/bootstrap_county.py --county "<Name>" --state "<State>" --slug <slug> --phase phase0` (recommended), **or** manually copy `config/counties/_template.json` to `config/counties/<slug>.json` and populate it. The template is intentionally not valid as a live county config until placeholders are filled.
5. Open Claude Code in the repo and paste this:
   ```
   Read MASTER_PROMPT.md and runs/<slug>/LAUNCH_<SLUG_UPPER>.md. Run Phase 0 only.
   ```
6. Claude Code runs Phase 0 → review change manifest → operator authorizes Phase 1 → Claude Code runs Phase 1 → review → and so on.
7. Run the deployment checklist in `MIGRATION.md` after Phase 8 completes.
8. The county refreshes on a schedule. It is not unattended: source heartbeats, review-queue depth, and adapter failures need an operator watching them, and portals change without notice. Budget for ongoing maintenance rather than assuming a finished build stays finished.

## County build workflow

The framework's build sequence, phase by phase:

1. **Run Phase 0: County Source Recon and Onboarding Gate** — Walk the exhaustive source-category checklist in `knowledge_base/domain/02_signals_and_sources.md` "Phase 0 source-category checklist". For each category: discover the URL by following official navigation, verify it's reachable, classify `official_status` and `lead_value`, set `source_priority` and `build_priority`, produce a portal fingerprint per `knowledge_base/engineering/00_tooling_decision_tree.md` Question 0, and capture `verification_note` and `open_questions`.
2. **Save verified source map to `config/counties/<county_slug>.json`** — The recon's output IS the populated county config. Copy from `_template.json`, populate every required field per source.
3. **Validate county config** — Run `python -m jsonschema config/counties/_schema.json config/counties/<county_slug>.json`. Must exit 0. P0 gate: at least one P0 source must be unblocked or have a committed unblock plan. `UNVERIFIED` and `NOT_FOUND` sources require `operator_override: true`.
4. **Run portal fingerprinting** — confirm `data/recon/<source_id>.fingerprint.json` exists for every source; adapter modules are selected from the fingerprint.
5. **Build one thin vertical slice** — Phase 1 synthetic harness → Phase 2 first adapter (usually parcel master enrichment) → Phase 3 first lead source. Prove one source end-to-end before scaling.
6. **Run tests** — `python scaffold/tests/run_all.py` must exit 0 (golden path + county-agnostic regression). Adapter fixture tests must pass per `engineering/05_verification_and_rollback.md`.
7. **Build remaining sources** — Phase 4 property matcher + review queue. Add additional adapters in `build_priority` order.
8. **Deploy dashboard** — Phase 5 dashboard customization → Phase 6 verification gate (mechanical verification + the §20 semantic verification contract; the production live-browser verifier is deferred to a future harness release) → Phase 7 refresh harness + alerts → Phase 8 `BUILD_SUMMARY.md`.

## How to run the gate tests

The framework ships **37 gate tests** that must all pass before a build is considered shippable. Run them with one command:

```
python scaffold/tests/run_all.py
```

The runner exits 0 only when every test exits 0. It auto-discovers everything in `scaffold/tests/v5_3_0/`, `v5_4_0/`, and `v5_6_0/`, so new invariants are gated without editing the runner. `v5_4_0_pending/` is deliberately NOT discovered — those are red-by-design behavioral specs.

Individual tests can be run directly for focused output:

```
python scaffold/tests/test_golden_path.py
python scaffold/tests/v5_6_0/test_recon_requires_freshness_check.py
```

## Versioning

This is **v5.6.0 (stable)**.

- Patch (5.0.1) — clarifications, doc fixes
- Minor (5.1.0) — new patterns, sources, deal paths, architecture additions
- Major (6.0.0) — breaking changes requiring migration of existing county builds

**v5.6.0 added** (released 2026-08-04):

Four mandatory recon steps (Protocol 01 §01.28–§01.32), each closing a class of false recon outcome observed in a real build, plus four new gate tests:

- **Gap 4 — access-control enforcement verification.** A control that *exists* is not a control that *blocks*. No source may be classified blocked until one low-volume good-faith request has been made and its actual response recorded. Adds `SINGLE_LAYER_HUMAN_VERIFIABLE` / `MULTI_LAYER` / `PER_REQUEST_CHALLENGE` tiering — a single-layer human-verifiable challenge is explicitly **not** a build blocker, it is an operator-assisted source cleared once by hand, after which the adapter resumes against the established session.
- **Gap 5 — canonical lead-type terminology verification.** Lead type names in §16.B are *framework* vocabulary, not local vocabulary. Terminology must be established empirically from the jurisdiction's own controlled vocabulary, and the **originating event** must be separated from downstream stages of the same distress process — the earliest reliably public artifact is where the lead-time advantage lives. Adds `NOT_APPLICABLE_IN_JURISDICTION` for types structurally absent under the local legal regime.
- **Gap 6 — tax roll and delinquency enrichment discovery.** `TAX_ROLL`, `DELINQUENCY_LIST`, and `BALANCE_LOOKUP` are searched and classified separately, with a delivery-mechanism preference order and a mandatory state-level fallback. A tax sale list covers only parcels already at sale eligibility and does not substitute for a delinquency feed.
- **Gap 7 — source freshness verification.** Advertised cadence is a *claim*; maximum actual record date is the *evidence*. Adds `LIVE` / `LAGGING` / `FROZEN` / `UNKNOWN`. A `FROZEN` source cannot satisfy the P0 gate regardless of record volume, and a stale bulk extract must never displace the live authoritative portal exposing the same records.
- 8 new locked rules in `FRAMEWORK_VERSION.json`.
- **README and `START_HERE.md` rewritten** to remove the "one sentence autonomous install" framing, which misrepresented how a county build actually runs.

**v5.5.0 added** (released 2026-06-26):

- **Source category expansion** — 15 mandatory Phase 0.A recon queries and 29 recognized lead types (#5).
- **Per-state lis pendens classification** — config-driven `state_rule_family` foreclosure regime, county-agnostic (#2).
- **PII guard for operator-authored county code** — gate test `test_no_pii_in_operator_code.py` + pre-commit hook + installer (#3).
- Added `IN_judicial_foreclosure` to the `state_rule_family` schema enum (#4).

**v5.4.0 added** (released 2026-06-25):

- **Executable §17–§20 staged pipeline** — normalize → classify → match → aggregate → score → review → dashboard. Monolith core retired.
- **Option-Y scoring seam** — `matched_leads.json` → scoring/dashboard cutover.
- **§20 semantic verification + evidence wire-through** (Session 5).
- **§19 idempotent aggregator** (Session 4).
- **9 deferred §17 debtor rules** implemented across 12 doc types; multi-owner contract extension; doc-type namespace bridge (R1/G1).

**v5.3.1 added** (released 2026-05-19):

- Removed the hardcoded county-slug default in `build_leads.py` and `verify_synthetic_harness.py`; both now auto-discover the active county config.

**v5.3.0 added** (released 2026-05-18):

- **§16 Source of Record Matrix** — the authoritative source for each field.
- **§17 Debtor Party Rules** — debtor-party identification rules.
- **§18 Signal Aggregation Contract** — combining signals across sources.
- **§19 Aggregator Idempotency Rule** — re-runs never duplicate or drift records.
- **§20 Semantic Verification Contract** — the semantic verification gate.
- **§13.14 enrichment-decoupling amendment** — enrichment is decoupled from lead origination.
- **§01 County Recon Protocol** upgrade and the new **§02 Build Mode Protocol**.
- 10 new framework invariants.
- Stub honesty disclosure for `verify_live.py` and `watchdog.py`.

**v5.2.0 added** (released 2026-05-15):

- **§13 Lead Origination Contract** — what events may originate a lead.

**v5.0.0 added** (released 2026-05-13):

- **Five-Layer Source Verification Gate** (MASTER_PROMPT Section 4.7) — every source goes through Official Origin → Source Category → Data Access → Lead Value / Source Role → Portal Proof verification before being trusted.
- **Source proof packet** — 18 new fields per source recording the verification outcome.
- **Build Eligibility Gate** (Section 4.10) — Phase 0 produces a `build_verdict` (`READY_TO_BUILD` / `READY_WITH_BLOCKERS` / `RECON_ONLY` / `WAITING_ON_ACCESS` / `NOT_BUILDABLE_YET`). Build Mode does not start without authorization.
- **Do Not Proceed Matrix** (Section 4.11) — 11 conditions that halt Phase 0 with a diagnostic verdict.
- **No False Dashboard rule** — a dashboard row is created by a lead event, never by a parcel record alone.
- **Source Hierarchy** (Section 4.9) — Tier 1 primary lead / Tier 2 supporting / Tier 3 enrichment. Only Tier 1 creates leads.
- **Recon Mode vs Build Mode** (Section 4.6) — first run is always Recon Mode.
- **VIP-friendly verdict message** (Section 4.12) — plain English Phase 0 output.
- **Operator-readable lead names rule** (Section 4.13) — no raw clerk codes in operator-facing surfaces.
- **Schema breaking changes** — 26 new source-level fields, 3 new top-level fields, 7 new enum types. v4.x configs require Phase 0 re-recon to populate new proof packet fields.

**v4.1.0 added** (preserved in v5.0.0): `scaffold/bootstrap_county.py`, `START_HERE.md`, `MASTER_PROMPT.md` Section 4.5, and the `runs/<slug>/` directory convention. The "one-sentence install" flow introduced here was retired in v5.6.0 — the tooling remains, the misleading framing does not.

Each county's `BUILD_SUMMARY.md` records the framework version it was built against.

## What this framework refuses to do

(From `domain/06_hallucination_controls.md` and the master prompt)

- Skip the live verification gate
- Ship leads without prime-directive labels (Confirmed / Estimated / Possible / Unknown)
- Declare a build done without `BUILD_SUMMARY.md` passing all checks
- Back-fill empty buckets with derived noise
- Generate leads from parcel-master metadata alone
- Mix synthetic data with real data in production `leads.json`
- Auto-merge entities when evidence is weak

## License

**Copyright © 2026 Xcerebro LLC. All rights reserved.**

This Framework is proprietary software, not open source. Use is governed by the terms in `LICENSE.md`. Access is granted only to active Xcerebro LLC VIP members and approved licensees.

Permitted: building county lead-generation systems for the licensee's own operations or for the licensee's paying client projects; modifying the Framework for internal or client-specific use.

Prohibited: reselling, redistributing, publishing, sublicensing, uploading to a public repository, sharing outside the VIP group, repackaging as the licensee's own product, or using the Framework to create a competing framework, course, or automation product.

See `LICENSE.md` for the complete terms, including revocation conditions and the no-warranty clause.
