# Atlas Creation Walkthrough — "Middle Georgia Atlas" for a non-technical user

*Written 2026-06-13, the session after the shared-dynamic-renderer migration was
marked code-complete. Purpose: stop trusting the "all 5 steps done" headline and
find what's **missing** to operate this as a real public platform — especially the
steps a non-developer cannot do themselves. The migration is real and works (I built
a whole atlas to prove it); the gap is that we have a working **engine with no car
around it**.*

> **Companion docs (read order):** HANDOFF.md (authoritative current state) →
> this doc → OPEN_GEOGRAPHIES_PLAN.md / ROADMAP.md (architecture & history, but
> **partly superseded** — their v0.8 "build/deploy pipeline" world was retired in
> the pivot; treat HANDOFF.md as truth). WIZARD_PLAN.md and DEMO_RUNBOOK.md also
> still describe pre-pivot Tina/build details in places.

---

## 0. What I actually did this session (evidence, not assumptions)

I created the Middle Georgia Atlas on the running local stack through the **same
backend the wizard drives** (`POST /core_data/atlases`), seeded it from GeoNames,
and reached it through the live shared renderer. Every ✅ below is something I
observed, not inferred.

| Step | Evidence |
|---|---|
| Create atlas | `POST /core_data/atlases` → **project 8 / site 7 / slug `middle-georgia`**, `live_url: null`, provision job 45 queued. Instant. |
| Provision | Job 45 → `completed` in <2s, single stage `search_index`. Typesense collection `middle_georgia_places` created (0 docs) + search-only key issued. |
| Atlas-by-slug API | `GET /core_data/public/v1/atlases/middle-georgia` → 200; branding.title "Middle Georgia Atlas", search app `places`, typesense block with scoped key, OSM base layer. |
| Seed (GeoNames) | Preview → **523 populated places** in the middle-GA bbox (Macon, Warner Robins, Milledgeville…). Import job 46 → `completed`, **523 imported / 0 failed / 0 skipped**. |
| Auto-index | `middle_georgia_places` → **523 docs** with no manual reindex (on-save `after_commit` indexing). Search "Macon" via the scoped search-only key → 2 hits (Macon, East Macon). |
| Live render | Through the **single running :4321 process** (default `OG_SITE_SLUG=jekyll-island-atlas`): `http://middle-georgia.localhost:4321/en/` → `<title>Middle Georgia Atlas</title>`; `/en/search/places` → **"523 results"** with real place names. Verified in real Chrome. |

**The headline finding:** the multi-tenant **code path is genuinely done**. A brand-new
atlas is queryable and renderable seconds after the wizard call, from one shared
process, with no build and no deploy — exactly as designed. Everything that's
missing is **around** that path: deploying it, giving it a URL, letting a stranger
sign up, letting them upload a logo, and the "bring your own WordPress" ask.

---

## 1. Executive summary — where the work actually is

Rough split of remaining effort by category (not by line count — by what blocks a
real public launch):

- **~60% Operations / Deployment** — none of this is running anywhere a stranger can
  reach. No renderer host, no Core Data prod deploy of the fork, no domain, no
  wildcard DNS/TLS, no managed Typesense, no S3 for media/tiles. **All blockers.**
- **~25% Product** — no self-serve signup, no media/logo upload, "bring your own
  WordPress" is unrealistic for the target user, no managed map-tile key story.
  **Mostly blockers for the *non-technical* persona specifically.**
- **~15% Code** — the migration is committed on **fork feature branches but not
  pushed/merged/released**; map cold-load & auto-fit warts; a few UX-friendliness
  gaps (cryptic GeoNames/Wikidata filters, Advanced-JSON-only knobs). **Mix of
  release-blocker and polish.**

A non-technical user, today, with a developer sitting next to them, can get an atlas
**provisioned, seeded, and rendering locally**. They cannot, by themselves, get a
**public URL anyone else can open**. That one sentence is the whole gap.

---

## 2. End-to-end walkthrough — the complete journey, per step

Persona: a humanities scholar in Macon with zero software expertise who wants the
"Middle Georgia Atlas" live for the public. Status key:
**✅ works · ⚠️ manual/confusing · ❌ missing · 🔧 needs a developer/ops.**

### Step 0 — Get a console account
- **User does:** Goes to the console and tries to sign in / sign up.
- **Where:** Console (`localhost:3000` locally; a hosted console URL in prod).
- **System:** Auth is **local JWT** (dev) or **Clerk SSO** (prod, if
  `VITE_CLERK_PUBLISHABLE_KEY` is set). Users are created by an **admin** via the
  admin-only Users section (`resources :users` is under `routes/admin.rb`).
- **Status: 🔧❌ No self-serve signup exists.** There is no "create account" /
  "request an atlas" flow in the client. A stranger cannot get in without an Emory
  admin manually creating their user (or a Clerk org invite). **First hard wall for
  the persona.** *(I logged in as the seeded `admin@example.com`.)*

### Step 1 — Start the wizard
- **User does:** Clicks "Create an atlas".
- **Where:** Console home / projects page → `/atlas/new`.
- **System:** Loads the 4-step wizard (`AtlasWizard.js`): basics → provision → seed → done.
- **Status: ✅** Entry point and stepper exist and gate on `canCreateProject`.

### Step 2 — Basics (name, slug, locale, template, area)
- **User does:** Types name "Middle Georgia Atlas"; optionally a slug; picks a locale;
  picks a template (**Places-only** vs **Atlas** = Places/People/Media/Topics); selects
  a geographic area.
- **Where:** Wizard step 1.
- **System:** Slug defaults to `name.parameterize`. Area is collected two ways
  (`AtlasAreaForm`), **mutually exclusive**:
  1. **Pick admin units** — cascading GeoNames dropdowns continent → country → state →
     counties (live GeoNames calls; `GEONAMES_USERNAME` is configured locally).
  2. **Draw a polygon** — `MapDraw` on a MapTiler base map (needs `VITE_MAP_TILER_KEY`
     + WebGL).
- **Status: ✅ mostly, ⚠️ in spots.**
  - ⚠️ **"Slug"** is jargon; a non-tech user won't know what it is or that it becomes
    the URL. (The auto-default saves them, but it's silent.)
  - ⚠️ **Area picker** works but the **admin-units cascade depends on live GeoNames**
    being reachable & credentialed (it is, on the hosted tier); the **draw map needs a
    MapTiler key + WebGL** (blank in headless). For Middle Georgia I used a drawn bbox.
  - ⚠️ Nothing explains the consequence of the area choice (it both frames the map and
    scopes the seed import). A non-tech user picks blindly.

### Step 3 — Provision
- **User does:** Clicks "Create"; watches a one-line stage checklist.
- **System:** `AtlasesController#create` **synchronously** creates Project (discoverable,
  caller = owner), template ProjectModels, SearchCollection, and Site (with default
  config: OSM layer + one `places` search app + locale). Then `ProvisionAtlasJob`
  creates the Typesense collection + search-only key. **The atlas is now live on the
  shared renderer — no repo, no build, no deploy.**
- **Status: ✅ excellent.** Instant in my run (<2s). This is the strongest part of the
  whole system.
  - ⚠️ **The "done"-adjacent promise of a URL is hollow locally:** `live_url` came back
    `null` because `OG_ATLAS_URL_TEMPLATE` is unset. Even when set
    (`https://{slug}.opengeographies.org`), that URL only resolves if the production
    routing in §3 is built — today it points at nothing. (See Step 12.)

### Step 4 — Seed places (authority import)
- **User does:** Toggles GeoNames (default on) and/or Wikidata; sets filters; clicks
  **Preview** (count + dots on a map), then **Import**.
- **Where:** Wizard step 3 (`PlaceImportPanel`), also re-runnable later as a
  project-scoped page.
- **System:** Per-source background Job; GeoNames queries by admin-unit or bbox;
  imports Places + names + geometry + a `web_identifier` (provenance/dedupe);
  auto-indexes after.
- **Status: ✅ works really well, ⚠️ filters are cryptic.**
  - ✅ I imported **523 populated places, 0 failed**, auto-indexed to 523 docs.
  - ⚠️ **GeoNames "feature classes"** are single letters (A/H/L/P/R/S/T/U/V) and
    **"feature codes"** are free-text (PPL, PPLA…); **Wikidata "types"** are Q-numbers
    (Q486972…). The defaults (GeoNames class **P** = populated places) are sensible, so
    a user who touches nothing gets a good result — but any customization needs domain
    knowledge a non-tech user lacks.
  - ⚠️ **Wikidata is disabled unless the area was *drawn*** (it queries by bbox); a user
    who picked admin units silently can't use it.
  - ⚠️ Preview map needs MapTiler + WebGL.

### Step 5 — Data entry & editing (custom records / fields / relationships)
- **User does:** Adds their own records, defines custom fields, draws geometries, links
  relationships beyond the seeded points.
- **Where:** The **standard Core Data console** (project → models / records / place
  editor).
- **Status: ✅ functional · ⚠️ this is the full power-user admin.** Defining
  ProjectModels, user-defined-field types, and relationship types — and using the
  place editor's map-draw (MapTiler) — is the real Core Data CMS, not a simplified
  surface. A non-tech user can edit a seeded record's name; building a custom data
  model is a developer/curator task.

### Step 6 — Search indexing
- **User does:** Nothing.
- **System:** On-save `after_commit` indexing keeps Typesense current; imports
  suspend+reindex in bulk; a manual "Reindex" button exists in the console.
- **Status: ✅ "just works."** 523 docs indexed with zero user action. This was a
  documented pain point (the old WAN rake job no non-developer could run) and it's now
  genuinely automatic.

### Step 7 — Configure the site (branding, nav, search/facets, layers, detail pages)
- **User does:** Opens the **Site edit page** — the durable config surface — and edits
  across 8 tabs: **General / Branding / Navigation / WordPress & Home / Map layers /
  Search apps / Advanced JSON / Config preview**.
- **System:** Writes the Site's JSONB config / branding / navigation; the renderer
  reads them by slug per request. Facets use **introspected pick-lists** from the
  project's fields (no UUIDs).
- **Status: ✅ comprehensive · ⚠️ a few sharp edges.**
  - ✅ Branding (title, fonts, colors, hide-title, allow-login), nav rows, search-app
    facet checkboxes, base/overlay layers, detail-page config — all in the UI.
  - ⚠️ **A freshly provisioned atlas is visually bare:** I saw the default dark-blue
    header, **no nav items, empty body, default fonts/colors**. It's *live* but looks
    unfinished until the user does this step. Nothing walks them through it.
  - ⚠️ The **Advanced JSON** tab (and some i18n knobs) expects raw JSON — a non-tech
    trap if they wander in.

### Step 8 — Connect WordPress + author home/standalone longform
- **User does:** In the **WordPress & Home** tab, pastes their WordPress host, sets a
  home tagline + a home WordPress page slug; creates WP pages with matching slugs for
  About/essays.
- **System:** Frontend fetches `/wp-json/wp/v2/{pages|posts}?slug=…` from **the user's
  own WordPress** and renders `content.rendered`; missing WP fails gracefully (empty).
- **Status: 🔧⚠️ The biggest *product* gap for this persona.**
  - The home **hero/tagline is console-native ✅**, but **all body longform requires the
    user to own and run a WordPress site**, create pages with exact slugs, and paste the
    host. There is **no managed WordPress, no console-native longform editor, and no
    validation** that the host/REST API is even reachable.
  - "Bring your own WordPress" is a reasonable ask for a digital-humanities *center*; it
    is **not** reasonable for a lone non-technical scholar. Expect most to either skip
    longform (bare site) or get stuck here.

### Step 9 — i18n text overrides (optional)
- **User does:** Edits common UI labels (Explore, Posts, Saved searches…) in the
  WordPress & Home tab's "Text overrides"; rarer keys via Advanced JSON.
- **Status: ✅ for the curated labels · ⚠️ long-tail keys are Advanced-JSON-only.**

### Step 10 — Map tiles for large datasets (PMTiles)
- **User does:** Clicks "Build map tiles" for big point sets.
- **System:** `BuildTilesJob` (tippecanoe) writes a `.pmtiles` archive and adds a layer.
- **Status: 🔧 Not wired for the new architecture in production.** In the old model the
  archive was copied into each site's static deploy dir; **there is no per-site static
  deploy anymore.** Generated tiles must be served from **S3/CDN via `TILES_PUBLIC_URL`**
  (absolute, range-capable) — which **isn't configured/hosted**. Locally it writes to a
  dir. *(Not needed for Middle Georgia's 523 points — clustering is fine below ~10k;
  required for OWA/GCA-scale sets.)*

### Step 11 — Upload a logo / media
- **User does:** Wants to put their atlas's logo in the header and images on records.
- **Status: ❌ No upload anywhere in the console.** The branding **logo is a plain text
  field** (`placeholder='/uploads/logo.png'`) — you paste a URL to an **already-hosted**
  image. There is **no file picker, no dropzone, no S3 presigned upload** in the Site
  editor. Record media goes through **IIIF Cloud** (separate, advanced). A non-tech user
  has **nowhere to put a logo image** they have on their laptop. **Hard wall for custom
  branding.**

### Step 12 — Actually view the live atlas at its URL
- **User does:** Opens the atlas URL.
- **Status (local): ⚠️ works, but only via the right mechanism.**
  - ✅ `http://middle-georgia.localhost:4321/en/` renders the branded atlas (home +
    523-result search) from the shared process via **Host-subdomain** resolution.
  - ⚠️ **`?atlas=<slug>` is a trap:** it sets the top-level page but **client island
    sub-fetches (`/config.json`, `/api/i18n`) drop the param** and fall back to the
    `OG_SITE_SLUG` default. I verified this directly — a `?atlas=` browse shows the right
    chrome but the **wrong atlas's search data**. Only **Host subdomain** (prod) or
    **`<slug>.localhost`** (local) or single-tenant `OG_SITE_SLUG` give a coherent
    session.
  - ⚠️ **Map cold-load wart:** on first paint the map was **blank**; it only rendered the
    base map after I clicked zoom, and even then sat at **world view with no auto-fit**
    to Georgia and unreliable cluster markers. The result **list** was always correct
    (523). A non-tech user's first impression is "my map is blank / shows the whole
    world with no pins." (Documented known issue: hidden-tab rAF suspension + missed
    MapLibre `load` + late auto-fit.)
- **Status (production): ❌ Missing entirely.** See §3 — there is **no deployed
  renderer, no domain, no wildcard DNS, no TLS**, so `https://middle-georgia.<domain>`
  resolves to nothing. The wizard's "live URL" is aspirational.

### Step 13 — Share it
- **Status: ❌ Blocked on Step 12 production routing.** No public URL exists to share.
  Once §3 is built, sharing is just the subdomain URL.

---

## 3. What's remaining — prioritized, separated by category

Ordered so that **earlier items unblock later ones**. Within each block: **🔴 blocker**
(no public launch without it) vs **🟡 polish** (launch is rough but possible).

### A. OPERATIONS / DEPLOYMENT (the bulk — all blockers)

| # | Item | Why it blocks | Notes |
|---|---|---|---|
| A1 🔴 | **Deploy the SSR renderer (Emory-hosted).** | It runs only as `node ./dist/server/entry.mjs` on localhost. | **No deploy artifacts exist** — no Dockerfile, compose, or service unit; no `start`/`serve` npm script; the `build` script still runs the retired `build.mjs`. Need: a Node host (ECS/EC2/VM), process manager, a build→release step (`SSR_ADAPTER=node astro build` on every frontend change), env (`OG_BASE_DOMAIN`, `OG_CONSOLE_URL`, Typesense creds). |
| A2 🔴 | **Own a base domain + wildcard DNS.** | The slug→URL model is `<slug>.<OG_BASE_DOMAIN>`. | Register/own e.g. `opengeographies.org`; add `*.<domain>` A/AAAA → renderer. Nothing exists today (the domain is only an *example string* in code). |
| A3 🔴 | **Wildcard TLS** for `*.<domain>`. | HTTPS for every tenant subdomain. | Let's Encrypt DNS-01 wildcard or Emory cert; reverse proxy (nginx/Caddy/ALB) terminating TLS → Node. Set `OG_BASE_DOMAIN` so the middleware derives slugs from Host. |
| A4 🔴 | **Deploy the Core Data fork to Emory prod.** | The whole platform (atlas-by-slug endpoint, `ProvisionAtlasJob`, place-import, discoverable security gate, console UI) lives in **our forks**, not in Emory's running Core Data. | Emory's prod DB is ~Dec-2025 schema (hence the `og_emory_schema_compat.rb` shim). Decide: land upstream in Performant **or** deploy the fork in Emory's layer. Until then, prod has none of this. |
| A5 🔴 | **Managed multi-tenant Typesense.** | Both renderer and console must reach one Typesense with per-atlas collections + search-only keys. | `typesense.ecds.io` is behind a bot wall today. Capacity/sizing/quotas/backups for N tenants unplanned (PLAN §7 open question). Key-issuance code exists. |
| A6 🔴 | **Set `OG_ATLAS_URL_TEMPLATE`** on the console once A2–A3 exist. | The wizard "done" step shows no link without it; users never learn their URL. | Trivial config — but meaningless until routing is real. |
| A7 🟡 | **Background-job runtime in prod (Sidekiq + Redis).** | Provision/import/reindex run on ActiveJob; dev uses in-process `:async`. | Prod needs Sidekiq + Redis so jobs survive and scale. |
| A8 🟡 | **Observability / ops runbook.** | Multi-tenant support load, error tracking, per-atlas health. | PLAN §7 "who operates what" still open. |

### B. MEDIA & TILES SERVING (mixed — blocker for rich atlases)

| # | Item | Cat | Notes |
|---|---|---|---|
| B1 🔴 | **S3 (or S3-compatible) bucket + CDN for media & PMTiles**, with `TILES_PUBLIC_URL` wired. | Ops+Code | The retired deploy pipeline's S3 object-sync logic is reusable, but **nothing serves media/tiles in the new architecture**. Range-capable host required for pmtiles. |
| B2 🔴 | **Logo/media upload UI in the console** (presigned S3 or IIIF Cloud), replacing the URL-paste logo field. | Code+Product | See Step 11 — currently no upload exists at all. |
| B3 🟡 | **Wire `BuildTilesJob` output to B1** and have the renderer consume absolute tile URLs. | Code | Job exists; only its prod destination is missing. |

### C. PRODUCT / ONBOARDING (blockers for the *non-technical* persona)

| # | Item | Notes |
|---|---|---|
| C1 🔴 | **Self-serve signup → project ownership.** | No registration flow; admin-invite only. Decide open vs gated; if open, wire Clerk signup → console user → `UserProject(owner)`. |
| C2 🔴 | **"Bring your own WordPress" alternative.** | Offer managed WordPress, OR a console-native longform editor, OR at minimum validate-and-guide the BYO-WP connect. Today longform is unreachable for a lone non-dev. |
| C3 🟡 | **Managed MapTiler (or key-free base map) story.** | Draw/preview/base maps need a MapTiler key. A shared Emory key with quotas, or an OSM-raster default, so users never paste a key. |
| C4 🟡 | **Multi-tenant authZ audit of the console.** | Confirm a non-admin owner sees/edits only their own project(s) across every console surface (jobs, imports, sites, descriptors). Wizard widened some policies; needs a full pass before strangers share the console. |
| C5 🟡 | **Friendlier seed filters + guided "configure your site" + slug explainer.** | GeoNames classes/Wikidata Q-numbers → presets; a post-provision checklist so the atlas isn't left bare. |

### D. CODE / RELEASE HYGIENE

| # | Item | Cat | Notes |
|---|---|---|---|
| D1 🔴 | **Push/consolidate/release the fork branches.** | Code | Migration is **committed but not pushed**: connector `og/automated-indexing` & cloud `og/automated-indexing` are **+1 ahead** of their fork remotes; places `og/introspected-config` has **no tracking branch**. Nothing is on a deployable release branch. |
| D2 🟡 | **Open the 6 upstream PRs** (UPSTREAM_PRS.md) + the `getCurrentURL` 7th candidate. | Code | Drafted, not opened. Independent of the pivot. |
| D3 🟡 | **`npm install` in core-data-places** to prune removed Tina packages + sync the lockfile before committing. | Code | `node_modules` still physically holds the deleted Tina deps. |
| D4 🟡 | **Keep local-only cloud wiring out of the release.** | Code | `Gemfile` path dep, `og_emory_schema_compat.rb`, `.gitignore` are dev-only; the fork branch must reference the connector via git and Emory's schema must be current (or keep the compat shim deliberately). |
| D5 🟡 | **Fix the `?atlas=` leak or remove it.** | Code | It half-works and will confuse anyone who finds it; client island fetches must carry the slug, or drop the param path and document subdomain-only. |
| D6 🟡 | **Map cold-load + auto-fit hardening** (`WhenStyleLoaded`, late-fit on user gesture). | Code | Pre-existing warts, but they're the literal first impression of every atlas. |

### E. SELF-HOST TIER (the ~2%, later)

| # | Item | Notes |
|---|---|---|
| E1 🟡 | **Docker compose + Terraform recipe** (renderer + Core Data + Typesense), same wizard pointed at a different Core Data URL. | ROADMAP v1.0 step 6; not built. Institutional/data-sovereignty tier only. |

---

## 4. Consolidated gap list — the things a non-developer literally cannot do alone

If you read nothing else, these are the **hard walls** for the target persona, in the
order they'd hit them:

1. **Get in the door** — no self-serve signup; an admin must create their account. *(C1)*
2. **Get a public URL** — the renderer isn't deployed, there's no domain, no wildcard
   DNS, no TLS; their atlas is reachable only on someone's localhost. *(A1–A3, A6)*
3. **Put their data live for real** — the platform code is in unmerged forks, not on
   Emory's production Core Data. *(A4, D1)*
4. **Upload a logo / images** — no upload exists; the logo field wants a URL to an
   already-hosted file. *(B2, B1)*
5. **Add longform / About pages** — requires owning and running their own WordPress and
   matching page slugs by hand. *(C2)*
6. **Trust the first impression** — the map opens blank / world-view with no pins until
   they fiddle with it. *(D6)*

Everything **between** those walls — provisioning, seeding from authorities, automatic
search indexing, branding/nav/facets/layers config, the live multi-tenant render — is
**genuinely working today** (I exercised all of it building project 8). The platform is
an excellent engine. It needs the operational chassis, a front door, and a media
on-ramp before a non-technical user can drive it to a public address.

---

## 5. Suggested critical path (shortest route to "a stranger can launch a public atlas")

1. **A4 + A5** — deploy the fork to Emory Core Data + stand up managed Typesense.
   *(Without this, nothing else matters — there's no prod backend.)*
2. **A1 + A2 + A3 + A6** — deploy the renderer behind `*.<domain>` with wildcard TLS;
   set `OG_BASE_DOMAIN` + `OG_ATLAS_URL_TEMPLATE`. *(Now provisioned atlases have real,
   shareable URLs — the keystone payoff goes public.)*
3. **C1** — self-serve signup → ownership. *(Now strangers can create atlases.)*
4. **B1 + B2** — S3/CDN + a logo/media upload UI. *(Now they can brand it.)*
5. **C2** — decide and ship the longform answer (managed WP / native editor / guided BYO).
6. **D1–D4** — push/merge/release the code and open upstream PRs (do alongside #1).
7. **D5 + D6 + C3 + C5** — polish: `?atlas=` leak, map cold-load/auto-fit, managed
   MapTiler, friendlier seed presets + a post-provision "finish your site" checklist.
8. **B3 / A7 / A8 / E1** — tiles-to-S3 wiring, Sidekiq, ops runbook, self-host recipe.

*Local artifact left in place for inspection:* the Middle Georgia Atlas is live on the
running stack — **project 8 / site 7 / slug `middle-georgia`**, 523 seeded places —
reachable at `http://middle-georgia.localhost:4321/en/` (home) and `…/en/search/places`
(523 results). Discard anytime; it does not touch project 2 (Private Gate fixture) or
project 7 (Jekyll).
