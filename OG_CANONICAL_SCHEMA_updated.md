# Open Geographies — canonical schema (first full draft)

*A concrete proposal, to be reacted to and edited — not a finished spec. Derived by the method
in `OG_SCHEMA_FIELD_METHOD.md` (what GCA already indexes ∩ what schema.org says the type is,
validated against HRCGA) and the de-bespoking decisions in `OG_SCHEMA_ANALYSIS.md`. Field names
are proposals; the **structure** is the part worth arguing about first.*

## 0. How to read this

Two layers, defined together because they must agree:

- **Authoring layer** — the Core Data `project_models` the OG template creates (a `model_class`
  plus user-defined fields and relationships).
- **Index layer** — the Elasticsearch document the indexer emits and the renderer queries. This
  is the actual contract between the two halves of the platform.

Two tiers, in every model:

- **Fixed** — same field, same name, every atlas. The renderer depends on these by name; they go
  in the ES mapping.
- **Per-atlas** — user-defined fields the curator adds. Carried in a fixed *container* (§2) so
  arbitrary fields need no mapping changes.

Conventions below: `?` = optional, `[]` = multi-valued.

## 1. The common envelope

Every indexed OG document — of any type — carries these. A shared index holding several model
types is what makes one renderer able to query them uniformly.

| Field | Type | schema.org | Notes |
|---|---|---|---|
| `uuid` | keyword | — | Stable identity. From Core Data. |
| `slug` | keyword | `url` | URL key. Unique per model type per project. |
| `project_id` | keyword | — | **Tenant key.** Every query filters on it, server-side. Non-negotiable. |
| `model_type` | keyword | — | `place` / `media` / `work` / `person` / `organization` / `map_layer` / `term` / `tour` / `event`. Lets one index hold many types. |
| `model_id` | keyword | — | Which *atlas-specific* model produced this (GCA "Churches" vs "States"). Drives per-model result cards. |
| `model_name` | keyword | — | The curator's label ("Churches"). Display only. |
| `name` | text + keyword | `name` | Required. Primary display + search field. |
| `names[]` | text | `alternateName` | Alternate/multilingual names. GCA weights this heavily in search. |
| `description?` | text | `description` | Rich text. |
| `short_description?` | text | `disambiguatingDescription` | Card/teaser text. |
| `visibility` | keyword | — | `published` / `hidden`. Generalizes GCA's `suppress`. Filtered server-side alongside `project_id`. |
| `identifiers[]?` | object | `sameAs` | `{ authority, identifier }` — viaf / wikidata / geonames. GCA already does this well. |
| `date_modified` | date | `dateModified` | |
| `udf` | object | — | Per-atlas fields — see §2. |

## 2. Per-atlas fields — three mechanisms, chosen by the nature of the value

*Revision: a value drawn from a controlled set (HRCGA's Denomination) should be a
**relationship to a Taxonomy model**, not a typed-in field — so nobody retypes "Methodist" and
consistency is enforced. He's right, and Core Data already implements exactly this. An earlier
draft of this section proposed a parallel `udf.*` convention; it is replaced by the below, which
uses the machinery that already exists.*

### The decision rule

> **If two records could ever share the same value and consistency matters → taxonomy
> relationship. If the value is unique to the record → user-defined field.**

"Methodist" is shared by hundreds of churches → taxonomy. A church's Legacy ID or year built is
its own → UDF.

### The three mechanisms

**1. Taxonomy relationship — the default for any controlled vocabulary.**
The term is a real record: entered once, reused, renameable everywhere at once, and able to carry
its own description, hierarchy (GCA's "Sub Topics"), and relationships. HRCGA's Denominations,
Categories, Counties; GCA's Types, Topics, Map Categories.

Core Data already indexes this. `Search::Base#build_relationships` writes every relationship under
the **relationship's UUID**, and `Search::Taxonomy` declares `search_attribute :name, facet: true`
— so each related term arrives as:

```
"<project_model_relationship_uuid>": [
  { "uuid": "…", "record_id": "…", "name": "Methodist", "name_facet": "Methodist", "inverse": false }
]
```

The facet path is therefore `<relationship_uuid>.name_facet`. No new convention needed.

**2. `Select` UDF — a small, closed enum.**
For a fixed list that will never need a description, hierarchy, or reuse as an entity (GCA's OSM
`Type`: node / relation / way). Options live on the field definition. Auto-faceted.

**3. Scalar UDF — genuinely per-record values.**
`String` / `Text` / `RichText` (free text), `Number`, `Date` / `FuzzyDate`, `Boolean`. Keyed by the
**field's UUID**, with `Date` / `Number` / `Select` / `Boolean` auto-faceted and text fields
opt-in via the indexer's `facet_field_uuids`:

```
"<user_defined_field_uuid>": 1878,
"<user_defined_field_uuid>_facet": 1878
```

### Promoted relationships — the part OG has to add

Core Data's UUID keys are correct but **opaque**: a shared renderer can't depend on
`"a3f1…c9"` meaning "the place type." So the canonical schema **promotes a small, fixed set of
structurally-required relationships to well-known names** in the index document — `types`,
`contained_in_place`, `media`, `works`, `people`, `places`, `map_layers` — which is what §§3–7
specify. Everything else stays under its relationship UUID.

That split is the whole trick:

- **Promoted** → the renderer depends on it by name, every atlas, no config.
- **UUID-keyed** → per-atlas, surfaced purely through config.

An atlas then declares its own facets by mapping a readable name onto the UUID path, so config
stays legible even though the wire format is UUIDs:

```json
"facet_attributes": [
  { "attribute": "denomination", "field": "<relationship_uuid>.name_facet", "type": "string" },
  { "attribute": "year_built",   "field": "<field_uuid>_facet",             "type": "numeric" },
  { "attribute": "types",        "field": "types",                          "type": "string" }
]
```

The renderer already speaks this dialect — `src/utils/search.ts` splits attributes into
relationship-id + field-id and resolves labels through i18n, which is why UUID keys work in the UI
today.

### Field intent — internal, displayable, promoted

*Addendum: HRCGA's Legacy ID was internal-only, never meant to be displayed —
and arbitrary UDFs are hard for a shared renderer to present meaningfully.*

The concern is concrete, not theoretical. The renderer today
(`src/apps/detailPages/UserDefined.astro`) iterates **every** key in `record.user_defined` and
renders it into a two-column grid, filtered only by an `excludes` array. That is exactly the
key/value dump — and it is **opt-out**, so a Legacy ID appears on the public page unless someone
remembers to exclude it. For a multi-tenant platform the failure mode (leaking an internal field)
sits on the wrong side, and it doesn't scale: every atlas must remember every exclusion.

So the schema carries **three levels of intent**, and display becomes **opt-in**:

| Intent | Meaning | Where it shows |
|---|---|---|
| **Internal** | Operational data — legacy IDs, import keys, reconciliation notes. | Nowhere public. Ideally not in the public index at all. |
| **Displayable** | Real content, but with no designed home of its own. | The curated attributes panel (below), if the atlas config places it. |
| **Promoted** | Structurally required; the renderer has designed UI for it. | Its own slot — title, hero media, map, related sections. |

**Where the intent lives — both layers, different jobs:**

- **`internal: true` on the field definition** (the model layer, alongside the existing `required` /
  `searchable` / `order`) is a **hard gate**: never indexed for public consumption, never rendered,
  regardless of config. This is what protects a Legacy ID by default. Best captured in the console
  at field-creation time, when the curator knows the intent.
- **The atlas config's display list** (the platform layer) is the **curation**: which of the public
  fields appear, in what order, with what label and format.

This split already has a precedent in the codebase: `searchable` is a property of the field
definition, while *facet* selection travels with the indexing options — `build_user_defined`'s own
comment notes that facet selection "is a publishing choice … rather than the shared field
definition." Display should follow the same seam.

### How the renderer presents arbitrary fields

The honest answer to "is a key/value table useful or pleasant?": **a dump is not; a curated,
ordered, labelled, type-formatted list is** — it is what every good museum or archive object page
is made of. The difference is entirely whether a human chose and ordered the fields.

The key insight for a multi-tenant renderer: **it never needs to understand a field's *meaning* —
only its *type* (how to format it) and its *slot* (where to put it).** That is sufficient to render
arbitrary fields well, generically. So the renderer offers a small fixed set of slots, and the
atlas config assigns fields to them:

| Slot | Filled by |
|---|---|
| Title / subtitle | `name`, `short_description` (promoted) |
| Hero media | `featured_media` (promoted) |
| Body | `description`, optional WordPress longform (promoted) |
| **Attributes panel** | An **explicit, ordered** list of UDFs the curator chose — label, value, formatted by data type |
| Related sections | `media`, `works`, `people`, `places`, `map_layers` (promoted) |
| Facets | Search only — the `facet_attributes` list |
| — | Everything else: **not rendered** |

The platform already has this pattern in the *search* config, where `result_card.attributes` is an
explicit ordered list carrying an `icon` and a `parser` per attribute. The detail page is the
outlier; making it opt-in the same way is a fix, not a new invention.

**Corollary:** this reinforces the previous section. Controlled values modelled as *taxonomy
relationships* get genuinely designed presentation (linked chips, term pages, facets), while scalar
UDFs land in the attributes panel. That is another reason to prefer relationships — they render
better, not just store better.

## 3. Place

The central model. `model_type: place`. An atlas may define **several** Place models (GCA:
Places + Counties; HRCGA: Churches + States) — one is designated primary via the Site's
`primary_place_model_id`.

| Field | Type | schema.org | Notes |
|---|---|---|---|
| `geo.point` | geo_point | `geo` (GeoCoordinates) | Centroid. **Required for map search** — the bbox filter runs on this. |
| `geo.shape?` | geo_shape | `geo` (GeoShape) | Real extent (polygon/collection) where known. |
| `geojson?` | object (not indexed) | — | The renderable FeatureCollection. Stored for display, not queried. |
| `bbox?` | double[4] | `spatialCoverage` | `[w, s, e, n]`. Drives zoom-to-extent. |
| `types[]?` | keyword | `additionalType` | From the atlas's own place-type vocabulary ("Barrier Island", "Church"). Faceted. |
| `contained_in_place?` | object | `containedInPlace` | `{ uuid, name, slug }`. **Replaces GCA's bespoke County relationship** — generalizes to county/state/region/parish. |
| `address?` | object | `address` (PostalAddress) | HRCGA needs this; GCA has no structured address. |
| `temporal_coverage?` | date range | `temporalCoverage` | Optional time span for the place itself. |
| `featured_media?` | object | — | One media ref for the hero slot. Generalizes GCA's separate `featured_photograph` / `featured_video`. |
| `media[]?` | object | `associatedMedia` | Denormalized media refs (§4 summary shape). |
| `works[]?` | object | `citation` | Related references. |
| `people[]?` | object | — | Related people. |
| `places[]?` | object | — | Related places (non-hierarchical). |
| `map_layers[]?` | object | `hasMap` | Related georeferenced overlays (§7). |

**Authoring layer:** `CoreDataConnector::Place`.

## 4. Media — one model for everything

`model_type: media`. This collapses GCA's three media models (Photographs as `MediaContent`,
Videos and Panos as generic `Item`s) into one, and adds audio.

| Field | Type | schema.org | Notes |
|---|---|---|---|
| `media_type` | keyword | (subtype) | `image` / `video` / `audio` / `pano` / `model3d`. Maps to ImageObject / VideoObject / AudioObject. |
| `content_url?` | keyword | `contentUrl` | **Uploaded** media (ActiveStorage, IIIF, or Mux playback URL). |
| `embed_url?` | keyword | `embedUrl` | **Externally hosted** (Vimeo / YouTube / Matterport). |
| `thumbnail_url?` | keyword | `thumbnailUrl` | |
| `iiif_url?` / `manifest_url?` | keyword | — | IIIF image / manifest, where applicable. |
| `encoding_format?` | keyword | `encodingFormat` | MIME type. |
| `duration?` | integer | `duration` | Seconds; audio/video. |
| `caption?` | text | `caption` | |
| `alt_text?` | text | — | Accessibility. HRCGA models this explicitly. |
| `content_warning?` | text | — | Already on Core Data's MediaContent. |
| `creator?` | object | `creator` | Person ref (HRCGA's "Photographer"). |
| `publisher?` | object | `publisher` | Organization ref (GCA's "Publisher"). |
| `upload_date?` | date | `uploadDate` | |
| `featured?` | boolean | — | Presentation hint, not structure. |

**Rule:** exactly one of `content_url` / `embed_url` is required. Hosting is a *field*, not a
model — Mux fills `content_url`, external embeds fill `embed_url`.

**Authoring layer:** `CoreDataConnector::MediaContent` — *including* video and pano, which is the
correction to GCA's `Item` modeling.

**Summary shape** when denormalized onto a Place: `{ uuid, slug, name, media_type, thumbnail_url,
content_url, embed_url, featured }`.

## 5. Work (references & bibliography)

`model_type: work`. GCA's "Works", HRCGA's "Resources".

| Field | Type | schema.org | Notes |
|---|---|---|---|
| `title` | text | `name` | (Envelope `name` carries it; `title` is the authoring label.) |
| `url?` | keyword | `url` | The "Link" field both atlases have. |
| `authors[]?` | text | `author` | |
| `publisher?` | object/text | `publisher` | |
| `date_published?` | date | `datePublished` | GCA uses a FuzzyDate "Year". |
| `page_start?` / `page_end?` / `pages?` | keyword | `pageStart` / `pageEnd` | |
| `volume?` / `issue?` | keyword | — | |
| `isbn?` | keyword | `isbn` | |
| `work_type?` | keyword | `additionalType` | From a per-atlas vocabulary (HRCGA's "Resource Types"). |
| `embeddable?` | boolean | — | Optional presentation hint (HRCGA). Not structural. |

**Authoring layer:** `CoreDataConnector::Work`.

## 6. Person & Organization

`model_type: person` / `organization`. Thin today in both atlases; schema.org supplies the
optional extras.

**Person:** `sort_name?`, `birth_date?` (`birthDate`), `death_date?` (`deathDate`), `job_title?`
(`jobTitle`), `affiliation?` (`affiliation`), plus envelope `name` / `description` /
`identifiers` (`sameAs`).

**Organization:** `url?`, plus envelope fields. GCA's "Publishers" is exactly this.

**Authoring layer:** `CoreDataConnector::Person` / `::Organization`.

## 7. Map Layer — first-class

`model_type: map_layer`. A georeferenced historical map overlay. Kept native to OG (a signature
GCA capability), generalized from "GCA's overlays" to "any atlas's map layer". **Topo Quads fold
in here** as layers that carry a date and a source.

| Field | Type | schema.org | Notes |
|---|---|---|---|
| `date?` | date | `temporalCoverage` / `datePublished` | **Drives the time-slider.** GCA's `topos` group by year. |
| `bbox?` | double[4] | `spatialCoverage` | Layer extent. |
| `geo.shape?` | geo_shape | — | Precise footprint where known. |
| `bearing?` | double | — | Georeferencing rotation (GCA has this). |
| `source` | object | — | `{ type: wms \| iiif \| xyz \| pmtiles \| annotation, urls[] }` — generalizes GCA's `wms_resources` and IIIF annotation pages into one typed source. |
| `preview?` | object | `associatedMedia` | Media ref — GCA's separate "Map Preview" model folds into §4. |
| `categories[]?` | keyword | — | Per-atlas vocabulary (GCA's "Map Categories"). |
| `publisher?` | object | `publisher` | |

**Authoring layer:** `CoreDataConnector::Place` (it carries spatial extent), as GCA already does.

## 8. Taxonomy term

`model_type: term`. The *mechanism* is fixed; the vocabularies are per-atlas.

| Field | Type | Notes |
|---|---|---|
| `vocabulary` | keyword | Which taxonomy ("Denominations", "Topics", "Types"). |
| `parent?` | object | Hierarchical terms (GCA's "Sub Topics"). |

**Authoring layer:** `CoreDataConnector::Taxonomy`.

## 9. Optional modules

Supported by OG, not required of every atlas.

- **Tour** (`model_type: tour`, `CoreDataConnector::Instance`) — schema.org `ItemList` /
  `TouristTrip`. `stops[]` as `{ position, place_ref }` (ordered), plus `thumbnail?` (media ref),
  `tour_type?`, `url?`. Consumed by the renderer *and* by Open Tour as an API client.
- **Event** (`model_type: event`, schema.org `Event`) — `start_date`, `end_date?`, `place_ref?`.
  **New** — neither GCA nor HRCGA has one, and it's the clearest gap for history-centric atlases.
  Proposed as optional rather than core until an atlas actually needs it.

## 10. Elasticsearch mapping sketch

The concrete contract. One shared index per model type (or one combined index), always filtered
by `project_id` server-side.

```json
{
  "mappings": {
    "dynamic_templates": [
      { "facets_as_keyword": {
          "match": "*_facet",
          "mapping": { "type": "keyword", "ignore_above": 256 } } },
      { "related_records": {
          "match_mapping_type": "object",
          "mapping": { "type": "object",
            "properties": {
              "uuid":       { "type": "keyword" },
              "record_id":  { "type": "keyword" },
              "name":       { "type": "text",
                              "fields": { "keyword": { "type": "keyword" } } },
              "name_facet": { "type": "keyword" },
              "inverse":    { "type": "boolean" } } } } },
      { "strings_as_text": {
          "match_mapping_type": "string",
          "mapping": { "type": "text" } } }
    ],
    "properties": {
      "uuid":              { "type": "keyword" },
      "slug":              { "type": "keyword" },
      "project_id":        { "type": "keyword" },
      "model_type":        { "type": "keyword" },
      "model_id":          { "type": "keyword" },
      "name":              { "type": "text", "fields": { "keyword": { "type": "keyword" } } },
      "names":             { "type": "text" },
      "description":       { "type": "text" },
      "short_description": { "type": "text" },
      "visibility":        { "type": "keyword" },
      "date_modified":     { "type": "date" },
      "identifiers":       { "type": "object",
                             "properties": { "authority": { "type": "keyword" },
                                             "identifier": { "type": "keyword" } } },

      "geo":   { "properties": { "point": { "type": "geo_point" },
                                 "shape": { "type": "geo_shape" } } },
      "geojson": { "type": "object", "enabled": false },
      "bbox":  { "type": "double" },
      "types": { "type": "keyword" },
      "contained_in_place": { "properties": { "uuid": { "type": "keyword" },
                                              "name": { "type": "keyword" },
                                              "slug": { "type": "keyword" } } },

      "media_type":   { "type": "keyword" },
      "content_url":  { "type": "keyword", "index": false },
      "embed_url":    { "type": "keyword", "index": false },
      "thumbnail_url":{ "type": "keyword", "index": false },

      "date":    { "type": "date" },
      "bearing": { "type": "double" },

      "media":      { "type": "object" },
      "works":      { "type": "object" },
      "people":     { "type": "object" },
      "places":     { "type": "object" },
      "map_layers": { "type": "object" }
    }
  }
}
```

Notes: `geojson` is `enabled: false` (stored, never indexed — it's payload, and indexing large
geometry twice is wasteful); URL fields are `index: false` (returned, never searched); `geo.point`
is what the map's bbox filter runs against.

## 11. Open questions

1. **One combined index or one per model type?** The renderer works either way (`model_type`
   filters within a combined index); Searchkick's per-model convention may favour separate ones.
2. **Which relationships get promoted** (§2) — the proposed set is `types`, `contained_in_place`,
   `media`, `works`, `people`, `places`, `map_layers`. Is that the right line between "renderer
   depends on it by name" and "per-atlas, config-driven"?
3. **How does the indexer know which relationship to promote?** A promoted relationship has to be
   identifiable per project — a naming convention on the relationship, a flag on the
   `project_model_relationship`, or a mapping declared in the Site config. This is probably the
   single most important unresolved mechanism.
4. **Do promoted relationships also keep their UUID key?** Emitting both is redundant but makes
   migration safe and keeps existing GCA facets working; emitting only the promoted name is
   cleaner. Suggest: emit both during migration.
5. **`geo.point` vs `geo.shape` as the map-search field** — centroid filtering is simpler and
   faster; shape is correct for large extents. Which does the map query run on?
6. **Does `contained_in_place` fully replace GCA's County relationship**, or does GCA need both
   for its existing facets?
7. **Is `Event` worth defining now**, or deferred until an atlas needs it?
8. **Should text UDFs be facetable by default?** Today `Date`/`Number`/`Select`/`Boolean`
   auto-facet and text is opt-in via `facet_field_uuids`. That opt-in is a publishing choice —
   does it live in the Site config now?
9. **Does `internal: true` belong on the user-defined field definition?** It is the clean way to
   keep a Legacy ID out of the public index and off the page by default, but it adds a column to
   the shared field model. The alternative — handling it purely in atlas config — is less safe
   (nothing stops an unconfigured atlas from leaking it).
10. **Should internal fields be excluded from the public index entirely, or indexed but never
   rendered?** Excluding is safer; indexing keeps them available for admin-side search and
   reconciliation. This may differ between the console's index and the renderer's.
