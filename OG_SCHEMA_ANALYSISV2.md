# Open Geographies — de-bespoking the GCA schema → OG final schema

*Working analysis for locking the schema. Reads the real GCA authoring schema
(`Georgia Coast Atlas schema.json`, 20 project models) and one runtime index document
(`sapelo.json`, an Elasticsearch place doc), separates the **universal core** from what's
**bespoke to GCA**, and cross-checks against schema.org to catch what OG's final schema is
missing. Neutral — meant to be worked over together and edited.*

## 0. Two layers, one exercise

The two files are different layers, and both need the same treatment:

- **Authoring schema** (`schema.json`) — the Core Data project *definition*: 20 `project_models`,
  each a `model_class` (Place/Person/…) + `user_defined_fields` + `project_model_relationships`.
  This is what the OG wizard/template must generate.
- **Runtime index doc** (`sapelo.json`) — the denormalized ES document the renderer queries
  (name, geo, and nested related photographs/videos/panos/works/topos/map_layers/places).

The whole job is the same on both: **freeze what's structural and universal, push what's
atlas-specific into per-atlas config, and drop GCA-only modules to optional add-ons.**

## 1. The 20 GCA models, sorted

By Core Data base class: **5 Place** (Places, Counties, Topo Quads, Map Layer, Map Features),
**3 MediaContent** (Photographs, Map Preview, Tour Thumbnail), **3 Item** (OSM, Videos, Panos),
**5 Taxonomy** (Types, Topics, Map Categories, Video Type, Tour Type), **1 each** Person
(People), Organization (Publishers), Work (Works), Instance (Tours).

| Model | Base class | Verdict | Why |
|---|---|---|---|
| **Places** | Place | **KEEP — the primary model** | Universal. Drop its `OpenStreetMap` + `KML` UDFs (import hacks). Keep Identifier / Short Description / Description. |
| **People** | Person | **KEEP** | General. Thin today; schema.org adds optional fields (§3). |
| **Works** | Work | **KEEP → standardize** | Bibliography/references. Real concept, but its 9 UDFs should map to schema.org `CreativeWork`. |
| **Photographs** | MediaContent | **KEEP → unify** | Correctly a MediaContent. Folds into a single **Media** model (§2). |
| **Publishers** | Organization | **KEEP** | General `Organization`; "publisher" is a credit role, not its own type. |
| **Types / Topics / Map Categories / Video Type / Tour Type** | Taxonomy | **PARAMETERIZE** | The *mechanism* (controlled vocabulary → facets) is universal; the *specific vocabularies* are per-atlas config, not fixed schema. |
| **Videos** | **Item** ⚠ | **DE-BESPOKE** | Mis-modeled as a generic Item with `embed_id` + `provider` (Vimeo/YouTube). Should be **Media (VideoObject) with an `embedUrl`**. This is the core media fix. |
| **Panos** | **Item** ⚠ | **DE-BESPOKE** | Same problem — an Item with a `Link`. A 360° pano is media; should be **Media with an `embedUrl`** (media type `pano`). |
| **Counties** | Place | **GENERALIZE** | The idea of a *secondary/administrative* Place model is universal — but "County" is GCA's. Replace the bespoke `County` relationship with schema.org `containedInPlace` hierarchy (§3). |
| **Tours / Tour Type / Tour Thumbnail** | Instance/Taxonomy/MediaContent | **STRIP → optional** | The Open Tour hook. Fine as an *optional* ordered-stops pattern (schema.org `ItemList`/`TouristTrip`), not part of every atlas's core. |
| **Map Layer** | Place | **KEEP → generalize (first-class OG feature)** | The georeferenced historical-map overlay (bbox, bearing, WMS resources, IIIF, **date**). A signature GCA capability worth making native to OG — the time-slider of historical maps. Generalize it from "GCA's overlays" to "any atlas's georeferenced map layer." Also carries OG's **temporal** dimension (see §3). |
| **Map Preview** | MediaContent | **KEEP → fold into Media** | Just the thumbnail/preview image for a Map Layer — absorbed by the unified Media model (§2), related to its Map Layer. |
| **Map Categories** | Taxonomy | **PARAMETERIZE** | Per-atlas vocabulary for classifying map layers — same treatment as the other taxonomies. |
| **Topo Quads** | Place | **FOLD into Map Layer** | Georectified historical USGS topo sheets — a *type* of georeferenced map layer, not a separate model. Generalize into Map Layer rather than strip or keep standalone. |
| **Map Features** | Place | **CLARIFY** | Empty (no UDFs/relationships) in the export — confirm its purpose before deciding; likely per-atlas annotations or unused. |
| **OSM** | Item | **STRIP** | OpenStreetMap node/way/relation linkage — a data-integration detail, not schema. |

**Bespoke UDFs to drop even on kept models:** `OpenStreetMap`, `KML` (Places/Counties —
import hacks), `WordPress Content` (Topics — WordPress is handled by the longform integration),
`embed_id` + `provider` (Videos — replaced by the unified media `embedUrl`), `Suppress`
(generalize to a `visibility` flag).

## 2. The media fix (the headline)

GCA proves the platform can embed Vimeo/YouTube — but it does it by **scattering media across
three models and two base classes**: Photographs (`MediaContent`), Videos (`Item` +
`embed_id`/`provider`), Panos (`Item` + `Link`). That's the ad-hoc convention we flagged
earlier, made concrete.

**OG's move: one unified Media model** (aligned to schema.org `MediaObject`), covering every
type, with two hosting modes as first-class structural fields — not a fork per host:

| Field | Meaning | schema.org |
|---|---|---|
| `mediaType` | image / video / audio / pano / 3d | (subtype: ImageObject / VideoObject / AudioObject) |
| `contentUrl` | uploaded file (Mux for video) | `contentUrl` |
| `embedUrl` | externally hosted (Vimeo / YouTube / Matterport) | `embedUrl` |
| `thumbnailUrl` | poster / thumbnail | `thumbnailUrl` |
| `encodingFormat` | MIME type | `encodingFormat` |
| `caption` / `alt` | text | `caption` |
| `creator` / `publisher` | credit → Person/Organization | `creator` / `publisher` |
| `featured` | presentation hint | — |

This single shape absorbs GCA's Photographs, Videos, and Panos, closes the modeling seam,
and **adds `audio`** (which GCA has none of — HRCGA needs it). `contentUrl` vs `embedUrl` is
exactly the uploaded-vs-embedded split; **Mux fills `contentUrl`, embeds fill `embedUrl`.**

## 3. schema.org cross-check — what OG is missing

GCA's schema is home-grown; aligning to schema.org makes OG's core standards-based (and makes
its identifiers/exports interoperable). Mappings + the gaps worth adding:

| OG core model | schema.org type | What GCA has | **Gap to add** |
|---|---|---|---|
| Place | `Place` | name, names, geo (point + geojson), types, county, bbox | **`address`** (PostalAddress); **`containedInPlace` / `containsPlace`** (a real hierarchy — replaces the bespoke County relationship and generalizes to States/regions); `additionalType` |
| Identifiers | `sameAs` + `identifier` | `identifiers[]` (viaf/wikidata/geonames) — already good | Formalize as `sameAs` (URLs) so it's standard, not custom |
| Media | `MediaObject` (+ Image/Video/**Audio**Object) | Photographs, Videos, Panos (fragmented) | **`AudioObject`** (none today); `encodingFormat`; `uploadDate`; unified `embedUrl`/`contentUrl` |
| Work | `CreativeWork` (Book/Article) | Works (Title/Authors/Publisher/Year/…) | `datePublished`, `isbn`, `pageStart`/`pageEnd`, `citation` — standard citation shape |
| Person | `Person` | name only | optional `birthDate`/`deathDate`, `jobTitle`, `affiliation`, `sameAs` |
| Organization | `Organization` | Publishers | `url`, `sameAs` |
| Tour | `ItemList` / `TouristTrip` | Tours (Instance + stops) | `itemListElement` w/ `position` — standard ordered-stops |
| **Map Layer** | `Map` (a CreativeWork) | Map Layer (bbox, bearing, WMS, date) | `spatialCoverage` (bbox), `temporalCoverage` / `datePublished` (the layer's date), `associatedMedia` (its preview) — standardize, but the georeferenced-overlay specifics (bearing, WMS, IIIF) stay domain-specific to OG. No perfect schema.org type — that's fine. |
| — (temporal) | `Event`, `temporalCoverage` | Map Layer `date` only | broaden a **temporal dimension** onto Place/Work too, not just Map Layer; consider **`Event`** for atlases whose subject is history-of-events |

**Two highest-value additions:** (1) **`containedInPlace`** — a real place hierarchy replaces
GCA's one-off "County" relationship and generalizes cleanly (island→county→state, church→county,
etc.); (2) an explicit **temporal dimension** — GCA already carries time on **Map Layer** (the
year that drives the historical-map slider), so the move is to *generalize* that temporal thinking
onto Place/Work as well (`temporalCoverage` / `datePublished`), and consider `Event` for
event-centric atlases. Map Layer is the proof OG already values time — the gap is making it
first-class beyond map overlays.

## 4. Proposed OG final schema (the target)

**Fixed structural core — every atlas, schema.org-aligned:**

- **Place** — `uuid`, `slug`, `name`, `names[]`, `description`, `short_description`, `geo`
  (GeoCoordinates point + GeoShape/geojson), `bbox`, `types[]` (`additionalType`), `sameAs`
  (identifiers), `containedInPlace`, `dateModified`, `visibility`, `project_id` (tenant),
  featured-media refs.
- **Media** (unified `MediaObject`) — the §2 shape; one model for image/video/audio/pano/3d,
  `contentUrl` **or** `embedUrl`.
- **Map Layer** (`Map`) — a first-class georeferenced map-overlay model: bbox, bearing, WMS/IIIF
  source, **date** (drives the time-slider), preview via Media. Optional to *populate* per atlas,
  but natively supported by OG (renderer knows how to show it). Topo Quads fold in as a type.
- **Work** (`CreativeWork`) — standardized citation fields.
- **Person**, **Organization** — thin, schema.org-shaped.
- **Relationships** — place↔media, place↔work, place↔person, place↔place
  (`containedInPlace`/related), place↔map-layer, optional tour (`ItemList`).

**Per-atlas variation — config, not fixed schema:**

- Controlled vocabularies / facets (Types, Topics, Categories, Denominations, Counties, Map
  Categories…) — each atlas defines its own and which become facets (the `facet_attributes`
  convention).
- Which Place model is **primary** (`primary_place_model_id`); support multiple Place models.
- Which first-class features an atlas *uses*: map layers, tours, panos (supported by OG, but not
  every atlas populates them).

**Dropped from core → optional/clarify:** OSM linkage (strip), KML/OpenStreetMap import UDFs
(strip), WordPress-content UDF (WordPress integration handles longform), Map Features (clarify
). Note: **Map Layer is retained as first-class**, not dropped.

## 5. Runtime index doc (`sapelo.json`) — same split, quickly

- **Keep (structural):** `uuid`, `slug`, `name`, `names[]`, `description`, `short_description`,
  `types[]`, `location` (point), `geojson` (FeatureCollection), `bbox`, `identifiers[]`
  (→ `sameAs`), related media/places/people/works, **`map_layers`** (first-class overlays),
  `date_modified`, `project_id`.
- **Generalize:** `suppress` → `visibility`; `county` → `containedInPlace`; `featured_photograph`
  / `featured_video` → generic featured-media refs; `media_types[]` from the unified Media model;
  `topos` → folded into `map_layers` (a map-layer type).
- **Bespoke → optional/strip:** `manifests` (IIIF-specific — keep only if IIIF stays a core
  concern), `population` (a one-off Place UDF).

## 6. The three decisions this forces

1. **Media:** adopt the single unified `MediaObject` (image/video/audio/pano) with
   `contentUrl` + `embedUrl` — collapsing GCA's Photographs/Videos/Panos. *(This is the knot;
   untie it first.)*
2. **Place hierarchy:** replace the bespoke County relationship with schema.org
   `containedInPlace`, and confirm the primary-Place-model + multiple-Place-models pattern.
3. **Core vs. module line:** agree what's first-class-but-optional (Map Layer overlays — a signature
   GCA feature OG keeps native — plus tours, panos) vs. what's genuinely dropped (OSM, import-hack
   UDFs) vs. per-atlas config (taxonomies/facets). Map Layer stays; only the truly incidental goes.
