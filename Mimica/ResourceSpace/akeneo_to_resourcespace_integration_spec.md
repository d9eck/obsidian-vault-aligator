# Akeneo CE v7 → ResourceSpace Integration Specification
**Project:** Ceramic / Pottery Commerce Architecture  
**Scope:** Akeneo to ResourceSpace integration only  
**Version:** 1.0  
**Status:** Draft for implementation  
**Authoring basis:** Uploaded Akeneo family/variant workbook, uploaded channel/locale matrix, and the agreed target architecture using ResourceSpace + S3 + imgproxy.

---

## 1. Purpose

This document specifies how **Akeneo CE v7** should integrate with **ResourceSpace** so that:

1. Akeneo remains the **catalog truth** for families, family variants, axes, sellable entities, locales, channels, and required media slots.
2. ResourceSpace becomes the **media production workspace** where creators know exactly **what images/documents are required**, for which product or variant, and in which workflow state.
3. The integration automatically creates or refreshes ResourceSpace work packages when a product or product model is ready for media production.
4. ResourceSpace metadata stays aligned with Akeneo codes so approved assets can later be published back to S3 and written to Akeneo as **imgproxy origin keys**.

This document covers the **Akeneo → ResourceSpace** direction only. The reverse publishing flow is referenced where needed but is not specified in full here.

---

## 2. High-level operating model

### 2.1 System roles

- **Akeneo**
  - Source of truth for:
    - families
    - family variants
    - axes and option codes
    - product models and variant products
    - channels and locales
    - required media attributes per entity level
  - Decides when an item is **media-ready** and should enter DAM workflow.

- **ResourceSpace**
  - Source of truth for:
    - upload tasks
    - image production workflow
    - resource metadata needed to map an asset to an Akeneo entity and media slot
    - selection / approval of candidate files

- **S3 + imgproxy**
  - Not directly part of this integration spec.
  - Mentioned only because ResourceSpace-targeting metadata must eventually support downstream publishing.

### 2.2 Guiding principle

**Creators should never have to infer the shot list from catalog structure manually.**  
Akeneo must generate the work context and push it into ResourceSpace.

That means the Akeneo integration should create or refresh a **ResourceSpace collection** representing the media brief for:

- a **single product** for non-variant families, or
- a **root product model** for variant families.

---

## 3. Source Akeneo model snapshot

This spec is based on the uploaded workbooks and assumes the following structure.

### 3.1 Families and variant strategy

| Family code | Family variant code | Variant levels | Axis level 1 | Axis level 2 |
|---|---|---:|---|---|
| `ART_UNIQUE` | none | product only | n/a | n/a |
| `TABLEWARE_FLAT` | `FV_TW_FLAT_SIZE_GLAZE` | 2 | `size_code` | `glaze_primary` |
| `TABLEWARE_HOLLOW` | `FV_TW_HOLLOW_SIZE_GLAZE` | 2 | `size_code` | `glaze_primary` |
| `DRINKWARE` | `FV_DRINK_CAPACITY_GLAZE` | 2 | `capacity_code` | `glaze_primary` |
| `VESSELS` | `FV_VESSELS_SIZE_GLAZE` | 2 | `size_code` | `glaze_primary` |
| `TILES_DECOR` | `FV_TILE_FORMAT_GLAZE` | 2 | `tile_shape_size` | `glaze_primary` |
| `SETS_BUNDLES` | `FV_SET_GLAZE` | 1 | `glaze_primary` | n/a |

### 3.2 Media attributes currently modeled in Akeneo

The workbook defines the following media key fields and localizable alt fields:

- `img_main_key`
- `img_topdown_key`
- `img_side_profile_key`
- `img_in_hand_key`
- `img_in_context_key`
- `img_installed_key`
- `img_components_flatlay_key`
- `img_gallery_json`
- `alt_main`
- `alt_topdown`
- `alt_side_profile`
- `alt_in_hand`
- `alt_in_context`
- `alt_installed`
- `alt_components_flatlay`
- `alt_gallery_caption_json`

The Akeneo → ResourceSpace integration must use this model as the authoritative definition of **required asset slots**.

### 3.3 Locales

Configured locales from the uploaded matrix:

- `es_ES`
- `en_GB`
- `en_US`
- `fr_FR`
- `fr_CA`
- `de_DE`
- `it_IT`
- `pt_PT`
- `nl_NL`

### 3.4 Regions and measurement systems

| Region code | Region | Measurement system | Default locales |
|---|---|---|---|
| `ES` | Spain | Metric | `es_ES`, `en_GB` |
| `EUC` | EU Core | Metric | `en_GB`, `pt_PT`, `fr_FR`, `it_IT`, `de_DE`, `nl_NL` |
| `EUE` | EU Extended | Metric | `en_GB` |
| `UK` | United Kingdom | Metric | `en_GB` |
| `NA` | North America | Dual (Metric + US) | `en_US`, `fr_CA` |

### 3.5 Touchpoints

The uploaded matrix includes channels under these touchpoints:

- `ECOMMERCE`
- `ETSY`
- `INSTAGRAM`
- `PRINT_CATALOG`
- `PHYSICAL_STORE`

---

## 4. Integration goals

### 4.1 Goals

The integration must:

1. Create or update a ResourceSpace work package when catalog items become media-ready.
2. Make the required shot list explicit for creators and DAM managers.
3. Prefill or constrain ResourceSpace metadata so codes match Akeneo exactly.
4. Stay idempotent: repeated syncs must update the same collection instead of creating duplicates.
5. Support all current families, axes, locales, regions, and touchpoints.
6. Make later ResourceSpace → Akeneo key write-back deterministic.

### 4.2 Non-goals

This spec does **not** define:

- binary file publication to S3
- imgproxy URL signing or transformation rules
- Akeneo write-back API for published keys
- localized alt text management inside ResourceSpace

Localized alt text and gallery captions remain owned by **Akeneo/PIM users**.

---

## 5. Recommended architecture

```text
Akeneo CE v7
  └─ media-ready event / scheduled export
      └─ Integration service
          ├─ read family, variant structure, axis values, channels, locales
          ├─ compute expected slots
          ├─ create/update ResourceSpace collection
          ├─ sync option lists / mapping metadata
          └─ optionally attach brief manifest
               └─ ResourceSpace
                   ├─ collection for work package
                   ├─ creators upload into collection
                   ├─ DAM reviews and approves
                   └─ later publisher exports approved assets
```

### 5.1 Recommended integration style

Use an **integration service** between Akeneo and ResourceSpace rather than point-to-point logic in either app.

The integration service should:

- call Akeneo APIs
- call ResourceSpace APIs
- own idempotency
- compute expected shot matrices
- sync dynamic option lists
- generate collection descriptions / manifests
- record sync status and diagnostics

### 5.2 Recommended trigger model

Use either of these patterns:

1. **Event-driven**
   - Trigger when a product or product model is marked media-ready.

2. **Scheduled reconciliation**
   - Nightly or hourly job scans Akeneo for items in media-ready state and upserts ResourceSpace collections.

Best practice: support **both**.
- Event-driven gives responsiveness.
- Scheduled reconciliation heals missed events.

---

## 6. Media-ready trigger definition

### 6.1 Required business trigger

A product or product model should be pushed to ResourceSpace only when it is ready for content creation.

Implement a custom Akeneo boolean or state field such as:

- `media_ready = true`
or
- `media_brief_status = READY`

### 6.2 Recommended Akeneo trigger conditions

Trigger collection create/update when any of these change on a media-ready item:

- family
- family variant
- product model code
- SKU
- axis values
- enabled status
- collection line / display name
- media-relevant option values that affect the brief
- membership of variant children under a root product model
- required channels/locales / market availability if you use them to scope creative work

### 6.3 Trigger granularity

- `ART_UNIQUE`: trigger at **product level**
- All 2-level variant families: trigger at **root product model level**
- `SETS_BUNDLES`: trigger at **root product model level**, because one collection should brief the full bundle family and glaze variants

---

## 7. ResourceSpace entity strategy

## 7.1 Primary work package = collection

Use a **ResourceSpace collection** as the work package created from Akeneo.

Reason:
- collections are first-class groupings
- they can be created by API
- creators can upload directly to a collection
- DAM managers can review all assets for a product line in one place
- one collection can represent the complete shot brief for a product model

### 7.2 Collection scope rules

| Family code | Collection scope | Pattern |
|---|---|---|
| `ART_UNIQUE` | per sellable SKU | `MEDIA__ART_UNIQUE__{sku}` |
| `TABLEWARE_FLAT` | per root product model | `MEDIA__TABLEWARE_FLAT__{product_model_code}` |
| `TABLEWARE_HOLLOW` | per root product model | `MEDIA__TABLEWARE_HOLLOW__{product_model_code}` |
| `DRINKWARE` | per root product model | `MEDIA__DRINKWARE__{product_model_code}` |
| `VESSELS` | per root product model | `MEDIA__VESSELS__{product_model_code}` |
| `TILES_DECOR` | per root product model | `MEDIA__TILES_DECOR__{product_model_code}` |
| `SETS_BUNDLES` | per root product model | `MEDIA__SETS_BUNDLES__{product_model_code}` |

### 7.3 Why collections should be created from Akeneo

This is the key operational decision.

If collections are created manually in ResourceSpace:
- creators must guess which products need work
- variant families become error-prone
- media requirements drift from Akeneo
- SKU/axis coverage is hard to audit

If collections are created from Akeneo:
- creators receive work packages only when catalog items are ready
- expected slots are known from the family model
- product model / variant structure stays aligned automatically
- the DAM team works from catalog truth instead of ad hoc requests

---

## 8. Expected shot matrix by family

The integration service must compute the required shot list from family and level.

### 8.1 `ART_UNIQUE`

**Collection scope:** one collection per sellable product / SKU

Required:
- `main`
- gallery roles as needed:
  - `lifestyle`
  - `detail`
  - `hero`

Akeneo write-back target:
- `img_main_key`
- `img_gallery_json`

### 8.2 `TABLEWARE_FLAT`

**Collection scope:** one collection per root product model

Required at `COMMON`:
- gallery roles:
  - `lifestyle`
  - `detail`
  - `hero`

Required at `LEVEL_2` for each `size_code x glaze_primary` SKU:
- `main`
- `topdown`
- `side_profile`

### 8.3 `TABLEWARE_HOLLOW`

Required at `COMMON`:
- `lifestyle`
- `detail`
- `hero`

Required at `LEVEL_2` for each `size_code x glaze_primary` SKU:
- `main`
- `topdown`
- `side_profile`

### 8.4 `DRINKWARE`

Required at `COMMON`:
- `lifestyle`
- `detail`
- `hero`

Required at `LEVEL_2` for each `capacity_code x glaze_primary` SKU:
- `main`
- `topdown`
- `side_profile`
- `in_hand`

### 8.5 `VESSELS`

Required at `COMMON`:
- `lifestyle`
- `detail`
- `hero`

Required at `LEVEL_2` for each `size_code x glaze_primary` SKU:
- `main`
- `topdown`
- `side_profile`
- `in_context`

### 8.6 `TILES_DECOR`

Required at `COMMON`:
- `lifestyle`
- `detail`
- `hero`

Required at `LEVEL_2` for each `tile_shape_size x glaze_primary` SKU:
- `main`
- `installed`

### 8.7 `SETS_BUNDLES`

Required at `COMMON`:
- `lifestyle`
- `detail`
- `hero`

Required at `LEVEL_1` for each `glaze_primary` variant:
- `main`
- `components_flatlay`

---

## 9. ResourceSpace metadata model required for the integration

This section defines the metadata fields that must be configured in ResourceSpace so Akeneo and ResourceSpace stay aligned.

> Design rule: use **fixed list fields** wherever the value must match Akeneo-controlled codes exactly.

## 9.1 Field groups

Create these ResourceSpace metadata field groups:

1. **Targeting**
2. **Asset Slot**
3. **Usage Scope**
4. **Creative / Technical**
5. **Publishing / System**

---

## 9.2 Targeting fields

These fields identify the Akeneo entity that the asset belongs to.

| Field code | Label | Type | Required | Indexed | Advanced search | Visible to | Source of values | Display condition | Validation / rule |
|---|---|---|---|---|---|---|---|---|---|
| `family_code` | Akeneo family | Fixed list (single) | Yes | Yes | Yes | Creators, DAM managers | Static list | Always | Must match one of the 7 family codes |
| `family_variant_code` | Akeneo family variant | Text (read-only) | Auto | No | No | DAM managers, system | Derived from Akeneo | Hidden or read-only | Auto-populated by integration |
| `entity_level` | Akeneo entity level | Fixed list (single) | Yes | Yes | Yes | Creators, DAM managers | Static list | Always | Allowed values: `PRODUCT`, `COMMON`, `LEVEL_1`, `LEVEL_2` |
| `product_model_code` | Akeneo product model code | Text | Conditional | Yes | Yes | Creators, DAM managers | Akeneo | When `entity_level` in `COMMON`,`LEVEL_1`,`LEVEL_2` | Required for variant families |
| `sku` | Akeneo SKU | Text | Conditional | Yes | Yes | Creators, DAM managers | Akeneo | When target is sellable | Required for `PRODUCT`, `LEVEL_2`, and `SETS_BUNDLES` `LEVEL_1` |
| `size_code` | Size code | Fixed list (single) | Conditional | Yes | Yes | Creators, DAM managers | Synced from Akeneo options | Family-specific | Required for relevant size-based sellable variants |
| `capacity_code` | Capacity code | Fixed list (single) | Conditional | Yes | Yes | Creators, DAM managers | Synced from Akeneo options | `DRINKWARE` only | Required for relevant drinkware variants |
| `tile_shape_size` | Shape + nominal size | Fixed list (single) | Conditional | Yes | Yes | Creators, DAM managers | Synced from Akeneo options | `TILES_DECOR` only | Required for relevant tile variants |
| `glaze_primary` | Glaze / colorway | Fixed list (single) | Conditional | Yes | Yes | Creators, DAM managers | Synced from Akeneo options | For glaze-varying sellable variants | Required where glaze is axis |
| `collection_line` | Collection / line | Fixed list (single) | Recommended | Yes | Yes | Creators, DAM managers | Synced from Akeneo options | Where available | Helpful for search and shoot grouping |

### 9.2.1 Static option values

#### `family_code`
- `ART_UNIQUE`
- `TABLEWARE_FLAT`
- `TABLEWARE_HOLLOW`
- `DRINKWARE`
- `VESSELS`
- `TILES_DECOR`
- `SETS_BUNDLES`

#### `entity_level`
- `PRODUCT`
- `COMMON`
- `LEVEL_1`
- `LEVEL_2`

### 9.2.2 Display conditions

Configure ResourceSpace field display conditions as follows:

- `size_code`
  - show only when:
    - `family_code` in `TABLEWARE_FLAT`, `TABLEWARE_HOLLOW`, `VESSELS`
    - and `entity_level` in `LEVEL_1`, `LEVEL_2`

- `capacity_code`
  - show only when:
    - `family_code = DRINKWARE`
    - and `entity_level` in `LEVEL_1`, `LEVEL_2`

- `tile_shape_size`
  - show only when:
    - `family_code = TILES_DECOR`
    - and `entity_level` in `LEVEL_1`, `LEVEL_2`

- `glaze_primary`
  - show when:
    - target is the glaze-varying leaf or variant level

- `sku`
  - show when target entity is sellable:
    - `ART_UNIQUE / PRODUCT`
    - any `LEVEL_2`
    - `SETS_BUNDLES / LEVEL_1`

---

## 9.3 Asset Slot fields

These determine which Akeneo media slot the asset should eventually populate.

| Field code | Label | Type | Required | Indexed | Advanced search | Visible to | Validation / rule |
|---|---|---|---|---|---|---|---|
| `asset_role` | Asset role | Fixed list (single) | Yes | Yes | Yes | Creators, DAM managers | Must be one of the approved roles |
| `gallery_position` | Gallery position | Integer | Conditional | Yes | Yes | Creators, DAM managers | Required for gallery roles |
| `is_preferred` | Preferred image for slot | Boolean | Recommended | No | Yes | DAM managers | One preferred candidate per slot if multiple exist |
| `shot_notes` | Shot notes | Text area | Optional | No | No | Creators, DAM managers | Internal production notes only |

### 9.3.1 `asset_role` option values

Use these exact values:

- `main`
- `topdown`
- `side_profile`
- `in_hand`
- `in_context`
- `installed`
- `components_flatlay`
- `lifestyle`
- `detail`
- `hero`

### 9.3.2 Mapping from `asset_role` to Akeneo attributes

| ResourceSpace `asset_role` | Akeneo target |
|---|---|
| `main` | `img_main_key` |
| `topdown` | `img_topdown_key` |
| `side_profile` | `img_side_profile_key` |
| `in_hand` | `img_in_hand_key` |
| `in_context` | `img_in_context_key` |
| `installed` | `img_installed_key` |
| `components_flatlay` | `img_components_flatlay_key` |
| `lifestyle`, `detail`, `hero` | `img_gallery_json` |

---

## 9.4 Usage Scope fields

These describe where the asset may be used and whether it needs localization or region-specific treatment.

| Field code | Label | Type | Required | Indexed | Advanced search | Visible to | Validation / rule |
|---|---|---|---|---|---|---|---|
| `touchpoint_scope` | Touchpoints | Fixed list (multi) | Recommended | Yes | Yes | Creators, DAM managers | Select one or more touchpoints |
| `region_scope` | Regions | Fixed list (multi) | Recommended | Yes | Yes | Creators, DAM managers | Select one or more regions |
| `contains_text` | Contains embedded text | Boolean | Yes | No | Yes | Creators, DAM managers | Drives locale requirement |
| `locale_scope` | Locales | Fixed list (multi) | Conditional | Yes | Yes | Creators, DAM managers | Required when `contains_text = true` |
| `measurement_variant` | Measurement variant | Fixed list (single) | Conditional | Yes | Yes | Creators, DAM managers | Required for assets with measurement text or market-specific dimension overlays |
| `channel_codes_json` | Expanded channel codes | Text area (hidden) | Auto | No | No | System only | Integration-owned cache of exact Akeneo channels |

### 9.4.1 `touchpoint_scope` options
- `ECOMMERCE`
- `ETSY`
- `INSTAGRAM`
- `PRINT_CATALOG`
- `PHYSICAL_STORE`

### 9.4.2 `region_scope` options
- `ES`
- `EUC`
- `EUE`
- `UK`
- `NA`

### 9.4.3 `locale_scope` options
- `es_ES`
- `en_GB`
- `en_US`
- `fr_FR`
- `fr_CA`
- `de_DE`
- `it_IT`
- `pt_PT`
- `nl_NL`

### 9.4.4 `measurement_variant` options
- `NONE`
- `METRIC`
- `DUAL_METRIC_US`

### 9.4.5 Locale / measurement rules

- For plain product photography:
  - `contains_text = false`
  - `locale_scope` may be empty
  - `measurement_variant = NONE`

- For text-bearing assets:
  - `contains_text = true`
  - `locale_scope` must be set to the active locales for the intended channels
  - if North America is in scope and measurements are present, set `measurement_variant = DUAL_METRIC_US`

---

## 9.5 Creative / Technical fields

| Field code | Label | Type | Required | Indexed | Advanced search | Visible to | Notes |
|---|---|---|---|---|---|---|---|
| `background_style` | Background style | Fixed list (single) | Recommended | Yes | Yes | Creators, DAM managers | `WHITE`, `NEUTRAL`, `IN_CONTEXT` |
| `orientation` | Orientation | Fixed list (single) | Recommended | Yes | Yes | Creators, DAM managers | `SQUARE`, `PORTRAIT`, `LANDSCAPE` |
| `crop_focus_notes` | Crop / focus notes | Text area | Optional | No | No | Creators, DAM managers | Optional art direction aid |
| `photographer_credit` | Photographer credit | Text | Optional | Yes | Yes | DAM managers | Governance |
| `copyright_owner` | Copyright owner | Text | Optional | Yes | Yes | DAM managers | Governance |
| `shoot_date` | Shoot date | Date | Optional | Yes | Yes | Creators, DAM managers | Audit |
| `retouch_status` | Retouch status | Fixed list (single) | Optional | Yes | Yes | DAM managers | `RAW`, `EDITED`, `FINAL` |

---

## 9.6 Publishing / System fields

These fields support later publish and audit flows. Hide them from creators.

| Field code | Label | Type | Required | Visible to | Owner | Notes |
|---|---|---|---|---|---|---|
| `publish_key` | Published asset key | Text | Auto | DAM managers, system | Publisher | Final origin key written after publish |
| `publish_bucket` | Origin bucket | Text | Auto | DAM managers, system | Publisher | Diagnostics |
| `publish_status` | Publish status | Fixed list (single) | Auto | DAM managers, system | Publisher | `NOT_PUBLISHED`, `QUEUED`, `OK`, `FAILED` |
| `published_at` | Published at | Date/time | Auto | DAM managers, system | Publisher | Audit |
| `publish_error` | Publish error | Text area | Auto | DAM managers, system | Publisher | Retry diagnostics |
| `source_resource_id` | ResourceSpace resource id | Text | Auto | System | System | Useful in logs and callbacks |

---

## 10. Dynamic option list synchronization

Some ResourceSpace fields should not be hard-coded permanently because the option values come from Akeneo and may evolve.

### 10.1 Static lists (configure once)
- `family_code`
- `entity_level`
- `asset_role`
- `touchpoint_scope`
- `region_scope`
- `locale_scope`
- `measurement_variant`
- `background_style`
- `orientation`
- `retouch_status`

### 10.2 Dynamic lists (sync from Akeneo)
- `size_code`
- `capacity_code`
- `tile_shape_size`
- `glaze_primary`
- `collection_line`
- optionally other controlled vocabulary fields that should appear in DAM search or briefs

### 10.3 Synchronization rule

The integration service should periodically reconcile Akeneo option codes into ResourceSpace fixed-list values.  
Do not maintain these lists manually in both systems.

---

## 11. Akeneo outbound data contract

The integration service should normalize Akeneo data into a payload like the following before calling ResourceSpace.

```json
{
  "event_type": "MEDIA_BRIEF_UPSERT",
  "source": "akeneo",
  "family_code": "DRINKWARE",
  "family_variant_code": "FV_DRINK_CAPACITY_GLAZE",
  "collection_code": "MEDIA__DRINKWARE__PM_CAFE",
  "collection_name": "Media Brief — DRINKWARE — PM_CAFE",
  "collection_scope": "PRODUCT_MODEL",
  "product_model_code": "PM_CAFE",
  "product_model_label": {
    "en_GB": "Cafe Mug",
    "es_ES": "Taza Cafe"
  },
  "channels": [
    "ecommerce_es",
    "ecommerce_eu_core",
    "ecommerce_uk",
    "ecommerce_na"
  ],
  "regions": ["ES", "EUC", "UK", "NA"],
  "active_locales": ["es_ES", "en_GB", "en_US", "fr_CA"],
  "expected_slots": [
    {
      "slot_code": "COMMON__gallery__lifestyle__01",
      "entity_level": "COMMON",
      "product_model_code": "PM_CAFE",
      "required_role": "lifestyle",
      "required": true,
      "gallery_position": 1
    },
    {
      "slot_code": "LEVEL_2__350ML__SAND__main",
      "entity_level": "LEVEL_2",
      "product_model_code": "PM_CAFE",
      "capacity_code": "350ML",
      "glaze_primary": "SAND",
      "sku": "DRK-350-SAND",
      "required_role": "main",
      "required": true
    },
    {
      "slot_code": "LEVEL_2__350ML__SAND__topdown",
      "entity_level": "LEVEL_2",
      "product_model_code": "PM_CAFE",
      "capacity_code": "350ML",
      "glaze_primary": "SAND",
      "sku": "DRK-350-SAND",
      "required_role": "topdown",
      "required": true
    },
    {
      "slot_code": "LEVEL_2__350ML__SAND__side_profile",
      "entity_level": "LEVEL_2",
      "product_model_code": "PM_CAFE",
      "capacity_code": "350ML",
      "glaze_primary": "SAND",
      "sku": "DRK-350-SAND",
      "required_role": "side_profile",
      "required": true
    },
    {
      "slot_code": "LEVEL_2__350ML__SAND__in_hand",
      "entity_level": "LEVEL_2",
      "product_model_code": "PM_CAFE",
      "capacity_code": "350ML",
      "glaze_primary": "SAND",
      "sku": "DRK-350-SAND",
      "required_role": "in_hand",
      "required": true
    }
  ]
}
```

### 11.1 Notes on this contract

- `collection_code` is the idempotency key for ResourceSpace-side upserts.
- `expected_slots` is the real work brief.
- The integration should persist the same manifest in its own logs/store even if only a summary is pushed into ResourceSpace.

---

## 12. What gets created in ResourceSpace

## 12.1 Collection

For each media-ready item, create or update a ResourceSpace collection with:

- name
- stable code embedded in the name or description
- owner / creator assignment if available
- summary of family, axes, locales, channels, and expected slots

### Required collection description content

At minimum, the collection description should include:

- family code
- family variant code
- product model code or SKU
- human-readable product label
- region(s)
- active locales
- expected shot list
- whether any localized/text-bearing assets are expected
- timestamp of last sync from Akeneo

### Recommended pattern

```text
Media brief for PM_CAFE
Family: DRINKWARE
Variant family: FV_DRINK_CAPACITY_GLAZE
Regions: ES, EUC, UK, NA
Locales: es_ES, en_GB, en_US, fr_CA
Expected slots:
- COMMON: lifestyle, detail, hero
- LEVEL_2 per capacity_code x glaze_primary: main, topdown, side_profile, in_hand
Last synced from Akeneo: 2026-03-25T21:30:00Z
```

## 12.2 Optional brief attachment

Recommended but optional:
- generate a `brief_manifest.json` or PDF and attach it into the collection
- include full slot matrix for creators and DAM managers

This is helpful when the collection grows large or when external photographers use upload links.

---

## 13. ResourceSpace workflow design

Use the Advanced Workflow plugin for bespoke workflow states.

### 13.1 Recommended states

- `BRIEF_CREATED`
- `UPLOADED`
- `SELECTED`
- `RETOUCH_PENDING`
- `APPROVED_FOR_PUBLISH`
- `PUBLISHED`
- `REJECTED`

### 13.2 Meaning

| State | Meaning | Typical actor |
|---|---|---|
| `BRIEF_CREATED` | Akeneo created or refreshed the media brief | Integration |
| `UPLOADED` | Candidate files uploaded | Creator |
| `SELECTED` | Best candidate chosen | DAM manager |
| `RETOUCH_PENDING` | Retouching still needed | DAM manager / retoucher |
| `APPROVED_FOR_PUBLISH` | Final asset approved | DAM manager |
| `PUBLISHED` | Asset has been exported downstream | Publisher |
| `REJECTED` | Candidate rejected | DAM manager |

### 13.3 Workflow constraint

Do not allow assets to leave `UPLOADED` unless required targeting metadata is present.

---

## 14. User workflow

### 14.1 Content creator workflow

1. Open assigned ResourceSpace collection created from Akeneo.
2. Review expected slot list.
3. Upload images into the collection.
4. Fill in required metadata:
   - family
   - entity level
   - product model / SKU
   - relevant axis values
   - asset role
   - gallery position if needed
   - usage scope metadata where relevant
5. Move item to `UPLOADED`.

### 14.2 DAM manager workflow

1. Review uploads against expected slots.
2. Select or reject candidates.
3. Verify targeting metadata.
4. Approve final assets for publishing.

### 14.3 PIM user workflow

PIM users do **not** manage binaries in ResourceSpace.  
They remain responsible in Akeneo for:

- `alt_main`
- `alt_topdown`
- `alt_side_profile`
- `alt_in_hand`
- `alt_in_context`
- `alt_installed`
- `alt_components_flatlay`
- `alt_gallery_caption_json`

---

## 15. Collection upsert rules

The integration must be idempotent.

## 15.1 Create

Create a new collection only if no existing collection matches the deterministic code pattern.

## 15.2 Update

If the collection already exists, update the description / manifest when any of the following change:

- product model membership
- variant matrix
- new SKU added
- axis value changed
- channel / locale scope changed
- expected slot list changed
- product renamed
- collection line changed

## 15.3 Do not duplicate

Never create a second collection for the same `collection_code` unless there is an explicit archival / versioning policy.

---

## 16. Slot matrix generation rules

The integration service must generate the slot matrix as follows.

### 16.1 Common algorithm

For each Akeneo item in media-ready state:

1. Read family and family variant.
2. Determine collection scope.
3. Read all relevant child variants.
4. For each required media slot defined by the family workbook:
   - create one expected slot entry
   - include targeting metadata needed for ResourceSpace
5. Write or refresh the ResourceSpace collection.

### 16.2 Family-specific rules

#### `ART_UNIQUE`
Generate:
- one `main` slot per product
- optional gallery slots per product brief template

#### `TABLEWARE_FLAT`
Generate:
- common gallery slots once per root model
- `main`, `topdown`, `side_profile` for each `size_code x glaze_primary` sellable leaf

#### `TABLEWARE_HOLLOW`
Generate:
- common gallery slots once per root model
- `main`, `topdown`, `side_profile` for each `size_code x glaze_primary` sellable leaf

#### `DRINKWARE`
Generate:
- common gallery slots once per root model
- `main`, `topdown`, `side_profile`, `in_hand` for each `capacity_code x glaze_primary` sellable leaf

#### `VESSELS`
Generate:
- common gallery slots once per root model
- `main`, `topdown`, `side_profile`, `in_context` for each `size_code x glaze_primary` sellable leaf

#### `TILES_DECOR`
Generate:
- common gallery slots once per root model
- `main`, `installed` for each `tile_shape_size x glaze_primary` sellable leaf

#### `SETS_BUNDLES`
Generate:
- common gallery slots once per root model
- `main`, `components_flatlay` for each `glaze_primary` level-1 sellable leaf

---

## 17. Region, channel, and locale policy for DAM

### 17.1 Default rule

For plain product photography:
- treat the asset as locale-neutral and broadly reusable
- use usage scope mainly for distribution reporting and future filtering

### 17.2 Text-bearing asset rule

For assets containing embedded text:
- mark `contains_text = true`
- populate `locale_scope`
- use only locales active for the target channels
- set `measurement_variant` where measurements appear in-image

### 17.3 North America rule

If the asset contains measurements and targets `NA`:
- set `measurement_variant = DUAL_METRIC_US`

### 17.4 Region-to-locale mapping

| Region | Locales |
|---|---|
| `ES` | `es_ES`, `en_GB` |
| `EUC` | `en_GB`, `pt_PT`, `fr_FR`, `it_IT`, `de_DE`, `nl_NL` |
| `EUE` | `en_GB` |
| `UK` | `en_GB` |
| `NA` | `en_US`, `fr_CA` |

---

## 18. API implementation notes

### 18.1 ResourceSpace capabilities to use

The integration should use ResourceSpace features for:

- collection creation
- metadata field updates
- fixed-list option handling
- workflow actions where needed

### 18.2 Recommended ResourceSpace API usage pattern

- create collections through the collection API
- for fixed-list metadata values, resolve option/node IDs before setting values
- keep the OpenAPI description available to developers if the installation exposes it

### 18.3 Integration-owned lookup tables

The integration service should maintain:
- mapping from Akeneo family to ResourceSpace collection strategy
- mapping from `asset_role` to Akeneo media attribute
- mapping from region to locales
- mapping from family to expected slot generator

---

## 19. Security and permissions

### 19.1 ResourceSpace groups

Recommended groups:
- `CREATORS`
- `DAM_MANAGERS`
- `PUBLISHER`
- `PIM_VIEWERS`

### 19.2 Permissions model

- creators can upload to assigned collections and edit creator-owned metadata
- DAM managers can edit all targeting metadata and move workflow states
- only publisher / system integration can write publish diagnostics fields
- PIM users should usually have read-only or limited access in ResourceSpace

---

## 20. Logging, diagnostics, and retry behavior

The integration must log:

- Akeneo source identifiers
- ResourceSpace collection id
- collection code
- create vs update action
- expected slot count
- last sync time
- errors returned by either API

### 20.1 Retry behavior

Use retryable jobs for:
- collection creation/update
- option list synchronization
- brief attachment upload

### 20.2 Hard validation failures

Fail the brief sync if:
- family code is unsupported
- required product model code is missing
- required SKU is missing for sellable targets
- axis values are missing for leaf-level slot generation
- collection code cannot be computed deterministically

---

## 21. Acceptance criteria

The integration is ready when all of the following are true:

1. A media-ready Akeneo item results in exactly one correct ResourceSpace collection.
2. Variant families produce the correct slot matrix for all leaf variants.
3. ResourceSpace metadata fields are preconfigured and validated as specified.
4. Creators can identify required uploads without referring back to Akeneo.
5. Re-running the sync updates the existing collection instead of duplicating it.
6. Region / locale rules for text-bearing assets are enforced consistently.
7. DAM managers can search ResourceSpace by family, product model, SKU, glaze, size/capacity, role, region, and publish status.

---

## 22. Recommended implementation phases

### Phase 1 — foundation
- configure ResourceSpace fields
- configure workflow states
- build collection upsert integration
- generate collection descriptions and expected slot manifests

### Phase 2 — option sync and creator UX
- sync dynamic option lists from Akeneo
- add upload shares if needed
- add dashboards / saved searches for “awaiting upload”, “awaiting selection”, etc.

### Phase 3 — downstream publish
- implement ResourceSpace → S3 publish
- write back `*_key` attributes to Akeneo
- implement Medusa / Keystone key-to-imgproxy URL building

---

## 23. Open decisions to confirm before implementation

1. Which Akeneo attribute will be the formal trigger:
   - `media_ready`
   - `media_brief_status`
   - something else

2. Whether `img_gallery_json` should always be required in Akeneo for all families or only when gallery roles are actually used.

3. Whether ResourceSpace should store a full machine-readable slot manifest:
   - in collection description only
   - as attached JSON file
   - or both

4. Whether upload shares are needed for external photographers or agencies.

5. Whether any channel requires truly distinct image sets beyond text-bearing assets.

---

## 24. References

### Official Akeneo references
- Families and family variants: https://help.akeneo.com/serenity-build-your-catalog/30-serenity-manage-your-families-and-variant-families
- Attributes: https://help.akeneo.com/v7-build-your-catalog/v7-manage-your-attributes

### Official ResourceSpace references
- API overview: https://www.resourcespace.com/knowledge-base/api/
- Create collection: https://www.resourcespace.com/knowledge-base/api/create_collection
- API Webhooks: https://www.resourcespace.com/knowledge-base/plugins/api-webhooks
- OpenAPI: https://www.resourcespace.com/knowledge-base/developers/openapi
- Basic metadata field configuration: https://www.resourcespace.com/knowledge-base/resourceadmin/basic-configure-metadata-field
- Advanced metadata field configuration: https://www.resourcespace.com/knowledge-base/resourceadmin/advanced-configure-metadata-field
- Indexing metadata: https://www.resourcespace.com/knowledge-base/resourceadmin/indexing-metadata
- Advanced workflow plugin: https://www.resourcespace.com/knowledge-base/plugins/workflow

---

## 25. Appendix A — quick implementation checklist

- [ ] Configure ResourceSpace metadata fields from section 9
- [ ] Configure ResourceSpace workflow states from section 13
- [ ] Implement collection code generator
- [ ] Implement family-based slot matrix generator
- [ ] Implement Akeneo media-ready trigger
- [ ] Implement collection upsert in ResourceSpace
- [ ] Implement dynamic option synchronization
- [ ] Validate display conditions for axis fields
- [ ] Validate sellable-level required targeting fields
- [ ] Add integration logging and retry handling
