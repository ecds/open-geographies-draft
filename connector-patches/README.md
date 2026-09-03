# connector-patches — four general changes to core-data-connector that need a new home

These are the only changes the platform side made to the connector itself (as opposed to
the engine). They are deliberately **general** — none is Open-Geographies-specific — and
they were kept as a thin overlay on unmodified upstream so the fork could stay minimal.

With the standalone `core_data_connector` gem now deprecated upstream (its code folded into
`core-data-cloud` / FairData), these commits have no upstream to go to and the fork they
live on is not where ECDS's code runs. They need to be **re-homed** — most likely into
ECDS's `core-data-cloud` fork, where the connector code now lives — or, where feasible,
absorbed into the engine. Paths will differ (the connector is now a subtree of cloud), so
treat these as patches to adapt, not cherry-picks to apply blind.

| # | Patch | What it does | Engine depends on it? |
|---|---|---|---|
| 1 | `0001-Public-API-enforce-the-project-discoverable-flag` | Adds `DiscoverableProjectScope`; the unauthenticated public API only serves projects flagged discoverable. | **Yes** — `engine/app/controllers/core_data_connector/public/v1/atlases_controller.rb` includes `DiscoverableProjectScope` and checks `discoverable`. The by-slug atlas endpoint the renderer relies on does not work without this. |
| 2 | `0002-Authenticated-project-descriptors-endpoint` | Authenticated endpoint returning per-project descriptors (models/fields) for the console. | **Yes** — `engine/app/models/core_data_connector/site.rb` reads `descriptors`. |
| 3 | `0003-Search-flatten-GeometryCollections-so-places-render-` | Flattens `GeometryCollection` geometries in search output so multi-part places render on result maps. | No. Note: the v1 engine's `Place.each_geojson_feature` solves the same problem differently (one Feature per member geometry). Likely superseded on the v1 path; still relevant to the legacy search path. |
| 4 | `0004-Search-skip-orphaned-relationships-per-index-facet-o` | Indexer skips relationship rows whose record was deleted (instead of crashing the run); adds per-index facet opt-in for user-defined fields. | No — but the orphan-skip is a robustness fix any indexer wants. |

Patches 1 and 2 are the priority: the engine cannot be mounted on an instance that lacks
them. Patches 3 and 4 are worth reviewing against the v1 engine's own handling before
deciding whether they still apply.

Generated with `git format-patch` from the platform fork's `og/connector-base` branch
against upstream `master`.
