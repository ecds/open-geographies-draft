# Open Geographies — platform-side design draft

Working design documents and the canonical schema for **Open Geographies (OG)**, a
no-code, multi-tenant geospatial publishing platform built on Core Data / FairData for
Emory's ECDS. This is the platform/renderer side of the project (provisioning, multi-tenancy,
the shared SSR renderer, the schema as consumed by clients), shared here as a draft so it can
be read alongside the lower-layer engine,
[`ecds/core-data-connector-open-geographies`](https://github.com/ecds/core-data-connector-open-geographies/tree/v1)
(branch `v1`).

Code lives in separate repositories; this repo is documents and schema artifacts only.

## Start here

- `og_schema/` — the canonical schema artifacts: `canonical_template.json` (authoring layer),
  `es_mapping.json` (index layer), and a README with a worked example. The engine carries the
  authoritative copy at
  [`lib/core_data_connector_open_geographies/v1/`](https://github.com/ecds/core-data-connector-open-geographies/tree/v1/lib/core_data_connector_open_geographies/v1);
  this is a mirror.
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
