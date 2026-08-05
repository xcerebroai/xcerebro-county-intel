# START HERE

**First time using the Xcerebro County Intelligence Framework? Read this before your first run.**

You're holding a private repository containing the harness for building a county lead intelligence system. It does not produce leads itself — it is the structure, the contracts, and the protocol that a build follows.

---

## Set expectations first

**This is an operator-supervised, checkpointed build. It is not a one-command autonomous install.**

Earlier versions of this document promised that you could type one sentence and watch a county build itself. That was removed in v5.6.0 because it was misleading, and because operators who believed it produced bad county builds.

Here is the honest version:

- The framework automates heavily **within** each phase and deliberately **stops between** phases.
- Phase 0 (county source recon) is the longest and most interactive part of any build. It is also the part that determines whether everything downstream is worth anything. It will ask you questions. Budget real time for it — a serious county recon is hours, not minutes.
- The judgment calls that matter most cannot be delegated blind: whether a source is *actually* blocked or just looks blocked, whether a lead type exists in that jurisdiction under a different name, whether a feed advertised as daily is actually current. The protocol forces these checks; a human still reviews the answers.
- Portals change without notice. A finished county build is a maintained system, not a finished artifact.

What you genuinely get: a rigorous, repeatable protocol; 36 automated gate test suites (540 assertions); contract schemas that keep the pipeline honest; and a build that refuses to fake productivity when a county's sources don't support it.

---

## What you need

- **Claude Code** installed and authenticated (`claude --version` prints a version)
- **Python 3.12+** (`python --version`) — 3.11 works for the gate tests
- A **private GitHub repo** for the county you're targeting, where the framework will live
- Optional but recommended: `pip install jsonschema` for strict county-config validation

---

## The first run, step by step

### 1. Put the framework in your county repo

The framework lives *inside* the county build repo. The county repo is what you commit and deploy.

### 2. Verify the harness is intact

```
python scaffold/tests/run_all.py
```

Expect `RESULT: PASS` across 36 suites. If this fails before you've changed anything, stop and fix the environment — don't build on a broken harness.

Note: `scaffold/tests/verify_synthetic_harness.py` is an intentional stub that prints a redirect and asserts nothing. Its acceptance coverage moved into `test_golden_path.py` and the two staged-pipeline end-to-end tests, which the gate does run. Do not treat that file as the synthetic harness.

### 3. Bootstrap the run folder

```
python scaffold/bootstrap_county.py --county "<Name> County" --state "<State>" --slug <county>_<st> --phase phase0
```

Slug convention is `<county>_<state>` — lowercase letters, digits, underscores. This creates `runs/<slug>/` containing a launch file and an operator notes file. It creates nothing else — no config, no scrapers, no data.

Many county names recur across multiple states — some in more than two dozen. The state argument is what disambiguates the target. Always pass it.

### 4. Start Phase 0

Open Claude Code in the repo and give it the launch file:

```
Read MASTER_PROMPT.md and runs/<slug>/LAUNCH_<SLUG>.md. Run Phase 0 only.
```

### 5. Work through Phase 0 with it

Phase 0 is the **County Source Recon and Onboarding Gate**. It does NOT build scrapers, dashboards, or databases.

It runs 15 mandatory discovery queries, a 5-layer verification gate on every source, a 29-type lead sweep, and seven required gap-closing steps:

    §01.22  sample document inspection — inspect real source documents, not just
            the listing page, before deferring a source
    §01.23  documented API discovery — search for an API before settling on
            HTML scraping
    §01.24  bulk-data availability classification
    §01.29  access-control ENFORCEMENT verification — a control that exists is
            not a control that blocks; test before recording blocked
    §01.30  lead-type TERMINOLOGY verification — framework names are not local
            names; find the originating event, not a downstream stage
    §01.31  tax roll / delinquency / balance-lookup discovery
    §01.32  source FRESHNESS verification — advertised cadence is a claim, max
            actual record date is the evidence

Expect approval prompts for web search and web fetch against official `.gov`, `.us`, and recognized vendor portal domains. These are safe to approve broadly within the county repo.

Phase 0 produces a recon dossier and one **build verdict**: `READY_TO_BUILD`, `READY_WITH_BLOCKERS`, `RECON_ONLY`, `WAITING_ON_ACCESS`, or `NOT_BUILDABLE_YET`.

### 6. Actually review the output

This is a real review, not a rubber stamp. Before authorizing Build Mode, check:

- **Every blocked source was tested, not assumed.** §01.29 requires a recorded response to a good-faith request. "CAPTCHA present in the page" is not a blocked verdict.
- **Every "not found" lead type was checked against the county's own vocabulary.** §01.30 exists because jurisdictions name the same event differently, and because the framework's name for a lead type is not the county's name for it.
- **Every "live" source has a max record date.** §01.32 exists because extracts get republished on a schedule long after the upstream feed stops delivering — the catalog date advances while the data does not.
- **The parcel/property join key is nailed down with examples.** If downstream sources can't normalize to one key, the build cannot match leads to properties, and no amount of later work fixes that cheaply.

### 7. Authorize Build Mode explicitly

Phase 0 ends at a hard gate. Claude Code will not enter Build Mode without an explicit instruction. Implicit approval is not accepted.

Then build **one thin vertical slice first** — a single adapter proven end to end — before scaling to the remaining sources.

---

## If the slug is wrong

Say so before proceeding:

> The slug should be `<county>_county_<st>` instead of `<county>_<st>`.

`<county>_<state>` is the default convention, not a hard requirement.

---

## The product rule

**This framework is an OFFICIAL EVENT SOURCE-DRIVEN lead intelligence system.**

Leads come from event-based and distress-based sources: clerk and recorder records, court events (foreclosure, probate, civil judgments, evictions), tax distress, sheriff sales, liens, judgments, lis pendens, code enforcement events, and recorded notices.

Parcel data, GIS data, assessor data, owner data, and tax roll data are **ENRICHMENT ONLY**. They are never the headline and never become leads on their own.

If no verified primary event source is accessible, Phase 0 stops. **It will not fill the dashboard with parcel data to fake productivity.** A blocked primary event source means a blocked build — not a parcel viewer dressed up as a lead system.

---

## What Phase 0 will refuse to do

Recon is metadata-only. It will not create accounts, pay for anything, solve CAPTCHAs, use proxies, bypass `robots.txt`, bypass access controls, scrape records, or submit public records requests. When access can't be determined without one of those actions, the correct outcome is an `UNKNOWN` classification and an escalation to you.

Note that this is a *recon* restriction. In Build Mode the framework's locked rules do permit operator-seeded sessions, CAPTCHA solvers, and stealth browsers where the operator authorizes them — a single-layer human-verifiable challenge is an operator-assisted source, not a dead one.

---

## If something goes wrong

**Phase 0 reports a failed P0 gate:** the county has no working daily-refresh distress source. Read the manifest. Decide whether to escalate for an unblock path or pick a different county. Do not proceed past Phase 0 with a failed gate.

**Phase 0 marks every source NOT_FOUND:** far more likely a recon failure than a county with no online records. Navigate the county's own website manually before accepting that result.

**`CONFIG_WRITE_FAILED`:** the county config failed validation twice. The framework will not silently proceed past a config-write failure. Open `runs/<slug>/CONFIG_WRITE_FAILED.md`.

**`SCHEMA_VALIDATION_SKIPPED`:** `jsonschema` isn't installed. Not an error — JSON syntax validation still ran. `pip install jsonschema` if you want strict validation.

**Claude Code starts building a scraper, dashboard, or database during Phase 0:** stop it.
> Stop. You are outside scope. Phase 0 only. Do not build scrapers, dashboards, or databases yet.

---

## Next steps

- `MASTER_PROMPT.md` — the framework contract
- `MIGRATION.md` — the full phase-by-phase build sequence
- `README.md` — project overview, current state, and what is *not* built yet
- `knowledge_base/protocols/01_county_recon.md` — the Phase 0 protocol in full

---

**You are licensed under the Xcerebro LLC proprietary VIP license. See `LICENSE.md`.**
