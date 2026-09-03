# Open Geographies — Engine Extraction: Result

*Executed 2026-06-15. The backend
engine extraction (plan §5 steps 0–6) is **done and verified on the running stack**.
This records what was built, the one refinement made during execution, the verification
evidence, and what remains. The console SPA stays on the fork bridge (separate track),
as planned.*

---

## What was built

### 1. `open_geographies` Rails engine — `repos/open-geographies-engine/`

A **non-isolated, mountable** engine that extends Core Data in place (keeps the
`CoreDataConnector::` namespace and `core_data_connector_*` table names, as recommended
in plan §1). Contents:

- **25 added app files** lifted verbatim from the fork (Site/SearchCollection models,
  the admin atlases/sites/search_collections/place_imports controllers + the public
  by-slug atlas controller, serializers, policies, the GeoNames/Wikidata import
  services, provisioning/auto-index/import jobs).
- **2 squashed migrations** (`db/migrate/`) — down from the fork's 9 — creating only
  `core_data_connector_search_collections` and `core_data_connector_sites` in their
  final shape (the retired deploy-pipeline add-then-drop churn is gone). **Create-only;
  no upstream-table ALTER** → installs additively onto clean upstream Core Data.
- **Decorators** (`lib/open_geographies/decorators.rb`) applied via `config.to_prepare`:
  the OG-coupled subset of the 20 modified files, as clean reopens/prepends —
  AutoIndexable into the 9 searchable models, AutoIndexable::Nested into
  PlaceGeometry/PlaceName/Relationship, Job dispatch (the 4 OG job types), JobPolicy
  member-view, ImportCsvJob suspend-and-reindex, GeoNames/Wikidata bulk methods, and
  Typesense single-record + progress.
- **Routes** appended into `CoreDataConnector::Engine.routes` (engine.rb initializer),
  so the OG endpoints serve under the existing `/core_data` mount.
- `engine_name 'open_geographies'` → `rails open_geographies:install:migrations`.

### 2. Connector mirror overlay — branch `og/connector-base` (in `repos/core-data-connector`)

Clean upstream `master` (`acba721`) + **4 thin, general-purpose, upstream-bound
commits**, **13 files, zero schema, zero OG-model references**:

| Commits | What | Origin |
|---|---|---|
| gate ×2 (`8c41501`, `c0e5584`) | enforce the public-API `discoverable` flag + authenticated descriptors endpoint | the already-prepared `pr/public-api-discoverable-gate` |
| geometry (`bbcd013`) | flatten GeometryCollections so places render | `pr/flatten-geometry-collections` |
| search-base | skip orphaned relationships + per-index facet opt-in | authored from emory-production's `search/base` **minus** the OG `include AutoIndexable` line |

### 3. Cloud wiring — `repos/core-data-cloud` (dev working tree, uncommitted)

`Gemfile` now depends on **stock connector** (`path: '../core-data-connector'`, checked
out on the clean `og/connector-base`) **+ the engine** (`path: '../open-geographies-engine'`).
No cloud `routes.rb` change needed (the engine appends its own routes). Kept uncommitted
per the cautions (the path deps, `Gemfile.lock`, `.gitignore`, `og_emory_schema_compat.rb`).

---

## The one refinement to the approved plan (please note)

**Plan §2 said:** upstream the discoverable gate, and *"carry as a deletable engine
override meanwhile."* During execution I found that carrying the gate **as an engine
decorator** would mean `class_eval`-copying ~300 lines of security-critical `base_query`
/ relationship-scope code into reload-time blocks — because the gate is an *inline*
edit and `base_query` calls `super` into `NestableController` (a prepend can't re-gate
it cleanly). That is the fragile "wholesale re-fork" the plan itself flagged.

**So I carried the gate (and geometry-flatten, and `search/base`'s general fixes) on the
connector mirror as thin, upstream-bound commits instead of engine `class_eval`s.** The
split principle: **general, inline-edit changes with zero OG-model references → connector
overlay; OG-coupled additive hooks → engine decorators.**

Why this is sound (and arguably better):
- It is the *same* "offer upstream / carry meanwhile" posture, in its **safest** form —
  no transcription of security code, using the already-tested PR commits.
- **It addresses the core concern**, which was *schema* divergence: the
  connector overlay has **zero schema changes** and **zero OG-model references** (verified).
- Each commit is atomic, rebaseable, and **droppable** the moment upstream merges it.
- The engine stays clean: only clear reopens/prepends, no fragile method copies.

Net: connector = upstream `master` + *our pending general-purpose PRs* (non-schema);
engine = all OG-specific code + the (additive) schema. If you'd prefer the connector be
bit-pure `master` with the gate inside the engine, that's the alternative — say so and
I'll move it, accepting the `class_eval` fragility.

---

## Verification (on the running stack — not just reasoned)

Local stack: Rails `:3001` on the engine, PostGIS `:54334`, Typesense `:8108` (demo
containers), against the real demo DB (project 7 "Jekyll", project 2 "Private Gate").

**Integration — 47/47 checks passed** (`tmp/verify_engine.rb`):
- `Rails.application.eager_load!` **clean** — no Zeitwerk namespace/collision errors
  across the 25 engine files in the `CoreDataConnector` namespace.
- All engine classes load; the connector overlay provides the gate concern + descriptors
  + geometry normalizer.
- Every decorator applied: 9 searchable models include AutoIndexable; 3 nested models;
  Job dispatch + constants; GeoNames/Wikidata bulk; Typesense index_record/remove_record
  + progress; JobPolicy + ImportCsvJob prepends.
- All OG routes appended into the `/core_data` set.

**Live HTTP:**
- `GET /core_data/public/v1/atlases/jekyll-island-atlas` → **200**, `slug=jekyll-island-atlas`,
  `title="Jekyll Island Atlas"`, `project_ids=["7"]`, 2 nav items. (Engine controller →
  Site model → serializer → appended route, end to end.)
- **Discoverable gate** (connector overlay): project 7 (discoverable) → **10 places**;
  project 2 "Private Gate" (discoverable=false) → **0 places**; unknown slug → **404**.
  The gate holds; fixtures intact.

**Migrations:** `open_geographies:install:migrations` copies the **2 create-only**
migrations (`.open_geographies.rb`); content verified additive (only the 2 `create_table`s).

---

## Current repo state

- **engine** `repos/open-geographies-engine/` — new local files (not yet a git repo / not pushed).
- **connector** — branch **`og/connector-base`** checked out (clean overlay). `emory-production`
  (the old 57-file fork line) untouched as rollback. Nothing pushed.
- **cloud** — on `og/automated-indexing`; working tree carries only the dev wiring
  (uncommitted). The old fork OG migrations remain in the committed history (untouched).
- Rails server (PID from this session) left running on `:3001`; demo containers up.

---

## What remains (not done this session — by design or pending your call)

1. **Console SPA** — **DECIDED 2026-06-15: option (b) standalone OG console is
   the target; the client fork stays as the bridge** (plan §3). The fork keeps shipping
   now; the standalone curator console is a separate ~1–2 wk track off the critical path.
   Not built yet — the decision is recorded, the work is queued.
2. **Commit / push** the engine (as `terminusfilms/open-geographies-engine`) and the
   `og/connector-base` branch — **not done; that's a publish action, awaiting your OK.**
   When ready, cloud's release Gemfile would switch the two path deps to git refs.
3. **`EMORY_HANDOFF.md` §6 rewrites** — the WS3 / schema-reconciliation / release-line
   sections still describe the superseded fork-deploy model. Quick to do once you confirm
   the direction.
4. **Clean-mirror cloud finalization** — removing the 9 old fork migrations from cloud's
   committed line and committing the stock-connector + engine Gemfile (kept as dev wiring
   for now; emory-production preserved as rollback).
5. **The upstream PRs** — the connector overlay commits *are* the PR content; opening them
   on `performant-software` still awaits your explicit OK (cautions).
