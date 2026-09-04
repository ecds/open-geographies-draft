# Open Geographies — platform-side design draft

Working design documents and the canonical schema for **Open Geographies (OG)**, a
no-code, multi-tenant geospatial publishing platform built on Core Data / FairData for
Emory's ECDS. This is the platform/renderer side of the project (provisioning, multi-tenancy,
the shared SSR renderer, the schema as consumed by clients), shared here as a draft so it can
be read alongside the lower-layer engine,
[`ecds/core-data-connector-open-geographies`](https://github.com/ecds/core-data-connector-open-geographies/tree/v1)
(branch `v1`).

The platform-side code lives in its own repositories in this organization:

- [`ecds/open-geographies-engine`](https://github.com/ecds/open-geographies-engine) — the
  `open_geographies` Rails engine (the upper layer: wizard backend, provisioning, Site
  config/branding/navigation, authority imports, the by-slug public atlas API). Mounts on
  Core Data / FairData and depends on the lower-layer engine above.
- [`ecds/core-data-places`](https://github.com/ecds/core-data-places) — the shared multi-tenant
  SSR renderer, a fork of Performant Software's
  [`core-data-places`](https://github.com/performant-software/core-data-places) (Astro + Node)
  with full upstream history. Renders any atlas by slug at request time; the Elasticsearch/
  Searchkit search path is in progress.

This repository holds the design documents, the schema mirror, and the connector patches.

## Start here

- `og_schema/` — the canonical schema artifacts: `canonical_template.json` (authoring layer),
  `es_mapping.json` (index layer), and a README with a worked example. The engine carries the
  authoritative copy at
  [`lib/core_data_connector_open_geographies/v1/`](https://github.com/ecds/core-data-connector-open-geographies/tree/v1/lib/core_data_connector_open_geographies/v1);
  this copy is the platform side's working version (v0.2.1-draft) — it carries proposals not
  yet applied to the engine's copy (notably the `preview` → `preview_media` promote rename) and
  is reconciled with the engine's copy at each template convergence, never merged independently.
- `OG_CANONICAL_SCHEMA_updated.md` — the prose schema spec (envelope, per-atlas fields,
  Place / Media / Work / Map Layer, open questions).
- `OG_SCHEMA_FIELD_METHOD.md` — how a model's fields are derived (GCA ∩ schema.org, validated
  against HRCGA).
- `OG_SCHEMA_ANALYSISV2.md` — de-bespoking the Georgia Coast Atlas schema into the OG core.

## Also here

- `OG_SCHEMA_ANALYSISV2.md` — the model-by-model analysis of what in the Georgia Coast Atlas
  schema is universal, per-atlas, or bespoke.
- `ENGINE_EXTRACTION_RESULT.md` — how the platform's additions were extracted into an additive
  Rails engine installed on unmodified upstream Core Data.
- `ATLAS_CREATION_WALKTHROUGH.md` — the atlas provisioning flow, end to end.
- `Georgia Coast Atlas architecture.png` — the reference architecture the platform generalizes.

These are working documents from an evolving design; the schema artifacts in `og_schema/`
are the normative part.
