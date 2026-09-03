# Open Geographies — how to derive a model's fields

*A repeatable method for deciding the fields of a major OG model (Place, Media, Work, Person…),
with Place worked as the example. The point: the canonical schema's fields aren't invented —
they're triangulated from three sources, then split into two tiers. This recipe can be run
model-by-model to lock the schema.*

## The fields come from three sources

For any major model, the candidate fields are triangulated from:

1. **What GCA already indexes in production** — the empirical floor. The live Elasticsearch
   documents (e.g. a Sapelo place doc) show exactly which fields a working atlas carries. Every
   field GCA renders has already earned its place; there's no guessing whether Place "needs" a
   given field — a running atlas already answers it.
2. **schema.org** for the corresponding type (`Place`, `MediaObject`, `CreativeWork`, `Person`) —
   the standards ceiling. This surfaces what GCA didn't happen to need but a general platform
   should support (`address`, `containedInPlace`, `sameAs`). It's the "what are we missing" source,
   and it keeps OG's identifiers and exports interoperable.
3. **A second real atlas (HRCGA)** — the generalization check. Comparing GCA's model against a
   second project's equivalent shows which fields are truly universal versus which were
   GCA-specific. A field present in both is a strong core candidate; a field in only one is
   probably per-atlas.

## Then split the candidates into two tiers

Not every candidate field belongs in the schema. For each model, fields fall into two tiers —
and assigning the tier is the actual schema-lock work:

- **Fixed structural fields** — hardcoded in the canonical schema and the Elasticsearch mapping,
  because the shared renderer depends on them *by name*. Identical for every atlas.
- **Per-atlas user-defined fields (UDFs)** — added by the curator in the console; they vary per
  atlas and surface as facets. **These are deliberately *not* in the fixed schema.**

So the fixed fields are a deliberate decision (triangulated from the three sources); everything
beyond them is intentionally left open as per-atlas UDFs. A third bucket, **drop**, catches
import/plumbing fields that aren't real content.

## Worked example — Place

| Field | Tier | Source | schema.org |
|---|---|---|---|
| `uuid` | **fixed** | Core Data base | identity |
| `slug` | **fixed** | GCA (URL key) | `url` |
| `name` | **fixed** | Core Data base + both atlases | `name` |
| `names[]` | **fixed** | GCA (alt / multilingual names) | `alternateName` |
| `description` | **fixed** | both atlases (RichText) | `description` |
| `short_description` | **fixed** | both atlases | `disambiguatingDescription` |
| `location` (lat/lon) + `geojson` (shape) | **fixed** | Core Data geometry | `geo` (GeoCoordinates + GeoShape) |
| `bbox` | **fixed** | GCA | `spatialCoverage` |
| `types[]` | **fixed** | GCA (Types taxonomy) | `additionalType` |
| `identifiers[]` (viaf / wikidata / geonames) | **fixed** | GCA | `sameAs` |
| `containedInPlace` | **fixed (new)** | schema.org — replaces the bespoke County relationship | `containedInPlace` |
| `dateModified` | **fixed** | Core Data timestamp | `dateModified` |
| `visibility` | **fixed** | generalize GCA `suppress` | — |
| `project_id` | **fixed (new, required)** | OG multi-tenancy | — |
| featured-media refs | **fixed** | GCA `featured_photograph` / `featured_video` → generic | — |
| related media / works / people / map layers | **fixed relationships** | GCA + HRCGA | `associatedMedia`, etc. |
| `Address`, `Year built`, `Legacy ID`, `Denomination`, … | **per-atlas UDF** | HRCGA-specific | (whatever fits) |
| `OpenStreetMap`, `KML`, `population` | **drop** | GCA import hacks / one-off | — |

Reading the table top to bottom shows the rule in action: the **fixed** block is "what both
atlases use, plus what schema.org says a Place is"; the **per-atlas** block is "what one atlas
happened to need"; the **drop** block is "import plumbing, not real content."

## The same recipe for the other models

- **Media** — GCA's `panos` / `videos` / `photographs` fields (`embedUrl`, `contentUrl`,
  `thumbnailUrl`, `mediaType`…) → cross-check schema.org `MediaObject` (+ Image/Video/Audio
  subtypes) → fixed core is the unified media shape; per-atlas UDFs are things like a "Transcript."
- **Work** — GCA's Works UDFs (Title / Authors / Publisher / Year…) → schema.org `CreativeWork`
  (Book / Article) → fixed core is the standard citation set.
- **Person / Organization** — thin today; schema.org adds optional `sameAs`, `birthDate` /
  `deathDate`, `jobTitle`, `affiliation`, `url`.

## In one sentence

The fixed fields of each model are the **intersection of "what GCA already indexes" and "what
schema.org says the type is," validated against HRCGA** — and everything outside that intersection
is a per-atlas user-defined field, not part of the frozen schema.
