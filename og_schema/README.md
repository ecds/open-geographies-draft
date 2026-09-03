# og_schema — the OG canonical schema, as artifacts

*v0.2.0-draft — the machine-readable half of `OG_CANONICAL_SCHEMA_updated.md`. The prose doc is
the argument; these files are the proposal made concrete, in formats that can actually be used
and diffed.*

*0.2.0 (post-v1-engine): media field names bent to what `MediaContent#extras` actually emits —
`thumbnail_url`/`iiif_url` → `thumbnail`/`preview` (`content_url`, `manifest_url`, `embed_url`,
`media_type`, `duration` already matched); `address` flattened to a single display-text field
promoted from a new Places "Address" UDF, since `administrative_area` (added in the engine's v1) now
serves the structured locality/region/country faceting.*

| File | Layer | What it is |
|---|---|---|
| `canonical_template.json` | Authoring | The project_models the wizard creates — same format as the GCA project export (`Georgia Coast Atlas schema.json`), so the two diff directly. Drives `Atlases::Template`. |
| `es_mapping.json` | Index | The Elasticsearch mapping the indexer writes to and the renderer queries. Directly usable as a create-index body. |

## The `og` extension keys (and what they decide)

The template carries three OG-specific keys that resolve open questions from the prose doc:

- **`og.promote`** — answers *"how does the indexer know which relationship/field to promote?"*
  (open question #3): **declared in the template/model definition itself**, not inferred from
  names or added to Site config. A promoted relationship or UDF is written to the well-known
  envelope field (`types`, `media`, `contained_in_place`, `description`, `embed_url`, …) **in
  addition to** its UUID key (question #4: emit both during migration).
- **`og.intent`** — `internal` / `displayable` (question #9): internal fields are never written
  to the public index and never rendered, regardless of config. Display of displayable fields is
  opt-in via the atlas config's attributes list.
- **`og.optional`** — modules an atlas may skip (Map Layers, Tours, Work Types). The models are
  in the canonical template; the wizard can offer them as toggles.

## Sample document — Sapelo Island, re-rendered into this schema

Abridged (2 media, 1 work, 1 layer), for a concrete before/after against `sapelo.json`:

```json
{
  "uuid": "75fc9ef3-7a0b-4856-83dd-ea8c574eef5f",
  "slug": "sapelo-island",
  "project_id": "1",
  "model_type": "place",
  "model_id": "6",
  "model_name": "Places",

  "name": "Sapelo Island",
  "names": ["île de Sapelo", "Isla Sapelo", "Zapala Island", "サペロ島"],
  "description": "<p>Sapelo's ecosystems, from salt marshes to maritime forests…</p>",
  "short_description": "Sapelo Island is home to the Gullah/Geechee people…",

  "visibility": "published",
  "date_modified": "2024-12-17T14:54:13Z",
  "identifiers": [
    { "authority": "viaf", "identifier": "https://viaf.org/en/viaf/143125668" },
    { "authority": "geonames", "identifier": "https://www.geonames.org/4212416" }
  ],

  "geo": { "point": { "lat": 31.4525, "lon": -81.2606 } },
  "geojson": { "type": "FeatureCollection", "features": ["…"] },
  "bbox": [-81.3067, 31.3833, -81.2185, 31.5199],

  "types": ["Barrier Island"],
  "contained_in_place": { "uuid": "…", "name": "McIntosh County", "slug": "mcintosh-county" },

  "featured_media": {
    "uuid": "4ce533c5-…", "slug": "sapelo-island-flyover", "name": "Sapelo Island Flyover",
    "media_type": "video", "embed_url": "https://player.vimeo.com/video/223819814",
    "thumbnail": "https://vumbnail.com/223819814.jpg"
  },

  "media": [
    { "uuid": "4ce533c5-…", "name": "Sapelo Island Flyover", "media_type": "video",
      "embed_url": "https://player.vimeo.com/video/223819814",
      "thumbnail": "https://vumbnail.com/223819814.jpg", "featured": true },
    { "uuid": "7ee132db-…", "name": "University of Georgia Marine Institute", "media_type": "pano",
      "embed_url": "https://s3.us-east-005…/UGAMI_0001_20230503_fs/index.htm",
      "thumbnail": "https://s3.us-east-005…_hd_t.jpg", "featured": false }
  ],

  "works": [
    { "uuid": "…", "name": "Making Gullah: A History of Sapelo Islanders…",
      "url": "https://…", "work_type": "Book" }
  ],

  "map_layers": [
    { "uuid": "ff352212-…", "name": "Cabretta Inlet", "date": "1954",
      "source": { "type": "annotation", "urls": ["https://…/annotation-page/1954/Cabretta_Inlet"] } }
  ],

  "a3f10000-relationship-uuid-example": [
    { "uuid": "…", "name": "Hog Hammock", "name_facet": "Hog Hammock", "inverse": false }
  ]
}
```

What changed vs the live GCA doc, in one glance:

- `panos` / `videos` / `photographs` (three arrays, two conventions) → **one `media[]`** with
  `media_type` + `embed_url`/`content_url`; `featured_video`/`featured_photograph` → one
  `featured_media`.
- `county: "McIntosh County"` (bare string) → **`contained_in_place`** (a real ref).
- `suppress: "no"` → **`visibility: "published"`**; `location {lat,lon}` → **`geo.point`**
  (+ optional `geo.shape`).
- `topos` (year-grouped annotation layers) → folded into **`map_layers[]`** as dated layers with a
  typed `source`.
- **`project_id` + `model_type`** appear — the multi-tenant keys GCA never needed.
- The unnamed extras (`population`) → per-atlas UDF keys, not envelope fields.

## Status

Draft for review — the `og.*` mechanism proposals (promote / intent / optional) are the
parts that most need his sign-off, since they land in his engine's territory (the field and
relationship models, and the indexer).
