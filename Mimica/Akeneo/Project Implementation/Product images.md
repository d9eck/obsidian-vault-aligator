Got it — with **Akeneo storing only ImageKit paths/keys** (not full URLs), the design gets cleaner and more future-proof. Below is a configuration-ready blueprint you can apply directly in **ResourceSpace** and **Akeneo CE v7**, plus how **Medusa/Keystone** build final CDN URLs.

---

# 1) Canonical reference model (new approach)

## What is the canonical “key”?

Define **one canonical ImageKit path/key per published asset**, e.g.:

```
products/TABLEWARE_FLAT/PM_OCEAN/dinner/COBALT/main__9f3c2a7.jpg
```

- This same string is:
    
    - the **S3 object key** (recommended: keep S3 key == ImageKit key)
        
    - the **Akeneo attribute value**
        
    - the suffix Medusa/Keystone uses to construct the ImageKit URL
        

## How Medusa/Keystone build the final URL

You keep a single base in config:

- `IMAGEKIT_URL_ENDPOINT = https://ik.imagekit.io/<your_imagekit_id>/`
    

Then:

```
finalUrl = IMAGEKIT_URL_ENDPOINT + key + "?tr=..."
```

So if Akeneo stores:  
`products/DRINKWARE/PM_CAFE/350ml/SAND/main__ab12.jpg`

Medusa renders:  
`https://ik.imagekit.io/<id>/products/DRINKWARE/PM_CAFE/350ml/SAND/main__ab12.jpg?tr=w-1200`

**Why this is future-proof**

- You can change domain, CDN vendor, signing rules, or transformation presets **without re-writing Akeneo data** (only app config changes).
    
- Keys remain stable identifiers of assets.
    

---

# 2) System roles (final)

### ResourceSpace (private/internal)

- Upload, tag, approve
    
- Workflow + metadata authority for assets
    
- Knows which asset maps to which **Akeneo product model / SKU**
    

### S3 (published bucket)

- Stores only _published_ deliverables (storefront-safe)
    
- Object keys mirror ImageKit keys
    

### ImageKit

- Reads from S3 as origin
    
- Serves via CDN + transforms via URL params
    

### Akeneo CE v7

- Owns product structure + merchandising data
    
- Stores **ImageKit keys** + localized alt/captions
    
- Completeness rules ensure every sellable SKU has required images
    

### Medusa + Keystone

- Consume Akeneo product data
    
- Construct final URLs and apply transformation presets per view/context
    

---

# 3) Storage & naming standards (so everything stays deterministic)

## 3.1 S3 keys = ImageKit keys (recommended)

Use **one unified path** for both. That avoids translation layers and mistakes.

### Proposed key pattern (fits your families/variants)

```
products/{FAMILY}/{PRODUCT_MODEL}/{AXIS1}/{AXIS2}/{ROLE}__{HASH}.{ext}
```

- `{FAMILY}`: `TABLEWARE_FLAT`, `DRINKWARE`, etc.
    
- `{PRODUCT_MODEL}`: your Akeneo product model code (e.g., `PM_CAFE`)
    
- `{AXIS1}`: `size_code` / `capacity_code` / `tile_shape_size` etc.
    
- `{AXIS2}`: `glaze_primary` (or `finish` if you add it as axis)
    
- `{ROLE}`: `main`, `topdown`, `side_profile`, `in_hand`, etc.
    
- `{HASH}`: content hash (immutable caching; no purges required)
    
- `{ext}`: `jpg`, `png`, `webp`, `pdf` (docs)
    

**Immutable rule:** if you re-edit an image, you create a new key (new hash). The publisher updates Akeneo with the new key.

---

# 4) ResourceSpace configuration: metadata fields you must add

Design goal: a creator can tag assets once, and publishing becomes automatic and correct.

## 4.1 Required metadata fields (create as custom fields)

Create these fields in ResourceSpace (names here are exact “field codes” you can mirror in your publisher):

### A) PIM targeting (required)

1. **pim_target_level** (single select, required)  
    Values:
    

- `SIMPLE_PRODUCT` (for ART_UNIQUE)
    
- `ROOT_MODEL`
    
- `LEVEL1_MODEL`
    
- `SKU_VARIANT`
    

2. **pim_family** (single select, required)  
    Values:  
    `ART_UNIQUE`, `TABLEWARE_FLAT`, `TABLEWARE_HOLLOW`, `DRINKWARE`, `VESSELS`, `TILES_DECOR`, `SETS_BUNDLES`
    
3. **pim_product_model_code** (text, required when level ≠ SIMPLE_PRODUCT)  
    Example: `PM_CAFE`, `PM_OCEAN`
    
4. **pim_sku** (text, required when level = SKU_VARIANT or SIMPLE_PRODUCT)  
    Example: `DRINK-350-SAND`
    
5. **pim_axis1_code** (text, conditional)  
    Set to the axis name used in that family:
    

- `size_code` / `capacity_code` / `tile_shape_size` / etc.
    

6. **pim_axis1_value** (text, required when level is LEVEL1_MODEL or SKU_VARIANT and you have axis1)  
    Examples: `dinner`, `350ml`, `2x6`
    
7. **pim_axis2_code** (text, conditional)  
    Usually `glaze_primary` (or `finish` if you later add it)
    
8. **pim_axis2_value** (text, required when level = SKU_VARIANT and you have axis2)  
    Examples: `COBALT`, `SAND`
    

> If later you add `finish` as a second axis at Level 2 for tiles: treat it as `pim_axis3_code/value` in the same pattern.

### B) Role + ordering (required)

9. **media_role** (single select, required)  
    Use a fixed controlled vocabulary (do not let people invent roles):
    

- `main`
    
- `topdown`
    
- `side_profile`
    
- `in_hand`
    
- `in_context`
    
- `installed`
    
- `components_flatlay`
    
- `detail`
    
- `lifestyle`
    
- `hero`
    
- `packaging`
    
- `scale_reference`
    
- `doc_spec_sheet`
    
- `doc_install_guide`
    
- `doc_care_instructions`
    

10. **media_position** (integer, optional but recommended)  
    Used for galleries and tie-breaking.
    
11. **media_is_primary** (boolean, optional)  
    Only one should be true per target entity. If true, publisher writes it to “main” key attribute even if role is lifestyle, etc. (useful for one-offs).
    

### C) Publishing control (required)

12. **publish_state** (single select, required)
    

- `DRAFT`
    
- `READY_FOR_REVIEW`
    
- `APPROVED`
    
- `PUBLISHED`
    

13. **publish_channels** (multi select, required)
    

- `ECOMMERCE`
    
- `CMS`
    
- `SOCIAL`
    
- `PRESS`
    
- `WHOLESALE`
    

14. **publish_replace_policy** (single select, recommended)
    

- `APPEND` (add to gallery)
    
- `REPLACE_ROLE` (replace main/topdown/etc.)
    
- `REPLACE_ALL_FOR_TARGET` (rare; wipes and rewrites the entire set for that SKU/model)
    

### D) Rendering hints (optional but powerful)

15. **crop_focus_x** (number 0–1)
    
16. **crop_focus_y** (number 0–1)  
    (If you ever want consistent smart crops in storefront.)
    
17. **background** (single select)
    

- `WHITE`
    
- `NEUTRAL`
    
- `IN_CONTEXT`
    

### E) Rights / provenance (optional but recommended)

18. **copyright_owner** (text)
    
19. **photographer_credit** (text)
    
20. **license_usage_notes** (textarea)
    

## 4.2 What creators actually do in ResourceSpace

- Upload to ResourceSpace
    
- Fill:
    
    - Target mapping (family, model, sku, axes)
        
    - role + position
        
- Move asset to `APPROVED`
    
- Publisher detects and publishes to S3/ImageKit and writes keys into Akeneo
    

No one copies URLs anywhere.

---

# 5) Akeneo CE v7 configuration: attributes (keys + alt)

Since you’re storing **keys**, the Akeneo side should be:

- **non-localizable keys**
    
- **localizable alt text/captions**
    
- completeness required at the right variant level
    

## 5.1 Attribute group

Create an attribute group: **MEDIA**

## 5.2 Core attributes (create once, reuse per family)

Below are configuration-ready attribute specs.

### A) Variant/SKU-level role keys (Text)

These are the “hard requirements” for sellable items.

|Attribute code|Type|Localizable|Scopable|Notes|
|---|---|--:|--:|---|
|`ik_main_key`|Text|No|No|Stores ImageKit path/key|
|`ik_topdown_key`|Text|No|No|Optional by family|
|`ik_side_profile_key`|Text|No|No|Optional by family|
|`ik_in_hand_key`|Text|No|No|Optional by family|
|`ik_in_context_key`|Text|No|No|Optional by family|
|`ik_installed_key`|Text|No|No|Optional by family|
|`ik_components_flatlay_key`|Text|No|No|Optional by family|

### B) Alt text (Text, localizable)

|Attribute code|Type|Localizable|Scopable|Notes|
|---|---|--:|--:|---|
|`alt_main`|Text|Yes|No|Required for SEO/accessibility|
|`alt_topdown`|Text|Yes|No|Optional|
|`alt_side_profile`|Text|Yes|No|Optional|
|`alt_in_hand`|Text|Yes|No|Optional|
|`alt_in_context`|Text|Yes|No|Optional|
|`alt_installed`|Text|Yes|No|Optional|
|`alt_components_flatlay`|Text|Yes|No|Optional|

### C) Galleries (Text Area JSON written by publisher)

For common marketing images (root model), and optionally variant extras.

|Attribute code|Type|Localizable|Scopable|Notes|
|---|---|--:|--:|---|
|`ik_gallery_json`|Text area|No|No|Ordered list of keys/roles|
|`gallery_caption_json`|Text area|Yes|No|Optional localized captions per item|

**Recommended JSON structure** (publisher-owned, never edited by hand):

```json
[
  {"role":"lifestyle","key":"products/DRINKWARE/PM_CAFE/hero__aa11.jpg","pos":1},
  {"role":"detail","key":"products/DRINKWARE/PM_CAFE/detail__bb22.jpg","pos":2}
]
```

If you want per-item localized alt/caption, use a keyed list:

```json
[
  {"key":".../detail__bb22.jpg","caption":"Close-up of rim detail"}
]
```

### D) Documents (keys)

|Attribute code|Type|Localizable|Scopable|Notes|
|---|---|--:|--:|---|
|`ik_spec_sheet_key`|Text|No|No|For tiles/architectural|
|`ik_install_guide_key`|Text|No|No|For tiles|
|`ik_care_instructions_key`|Text|No|No|Shared or per family|

### E) Sync diagnostics (optional but highly recommended)

|Attribute code|Type|Localizable|Scopable|Notes|
|---|---|--:|--:|---|
|`media_last_published_at`|Text|No|No|ISO datetime string|
|`media_publish_status`|Simple select|No|No|`ok/pending/failed`|
|`media_source_ref`|Text|No|No|e.g., ResourceSpace resource id(s)|

---

# 6) Attribute distribution per your families/variants (exact placement)

## ART_UNIQUE (no variants)

**Entity:** Simple product  
Required:

- `ik_main_key`
    
- `alt_main`  
    Optional:
    
- `ik_gallery_json`
    

## TABLEWARE_FLAT (FV_TW_FLAT_SIZE_GLAZE)

- **Root model (COMMON)**:
    
    - `ik_gallery_json` (lifestyle/detail shared across the line)
        
- **Level 1 (size_code)**:
    
    - usually no keys
        
- **Level 2 (SKU, glaze_primary)**:
    
    - Required: `ik_main_key`, `ik_topdown_key`, `ik_side_profile_key`
        
    - Required alts: `alt_main` (and optionally the others)
        

## TABLEWARE_HOLLOW (FV_TW_HOLLOW_SIZE_GLAZE)

- Root: `ik_gallery_json`
    
- SKU: Required `ik_main_key`, `ik_in_hand_key` (your “scale perception” hero)
    
- Alt: `alt_main` required
    

## DRINKWARE (FV_DRINK_CAPACITY_GLAZE)

- Root: `ik_gallery_json`
    
- SKU: Required `ik_main_key`, `ik_side_profile_key`
    
- Alt: `alt_main` required
    

## VESSELS (FV_VESSELS_SIZE_GLAZE)

- Root: `ik_gallery_json`
    
- SKU: Required `ik_main_key`, `ik_in_context_key`
    
- Alt: `alt_main` required
    

## TILES_DECOR (FV_TILE_FORMAT_GLAZE)

- Root: `ik_gallery_json` (collection/installed inspiration can live here too)
    
- Level 1 (tile_shape_size):
    
    - optional docs at this level if they vary by format:
        
        - `ik_spec_sheet_key`, `ik_install_guide_key`
            
- SKU: Required `ik_main_key`, `ik_installed_key`
    

## SETS_BUNDLES (FV_SET_GLAZE)

- Root: `ik_gallery_json`
    
- SKU: Required `ik_main_key`, `ik_components_flatlay_key`
    
- Alt: `alt_main` required
    

---

# 7) Completeness rules (Akeneo configuration you should enforce)

In each family, mark required at the **sellable level** (SKU or simple product):

- All families: `ik_main_key`, `alt_main`
    
- Plus family-specific requirements above (e.g., flatware requires topdown + side_profile)
    

This gives you a simple “ready for ecommerce” completeness gate.

---

# 8) Publisher logic (how keys are generated and written)

## Inputs (from ResourceSpace metadata)

- Target entity (SKU/model + axes)
    
- `media_role`, `media_position`
    
- `publish_channels`
    
- approved state
    

## Outputs

1. Upload to S3 under generated immutable key (pattern above)
    
2. Write to Akeneo:
    

- role key attribute (`ik_*_key`) for required roles
    
- append to `ik_gallery_json` for gallery roles (`lifestyle`, `detail`, `hero`, etc.)
    

### “Replace role” behavior

If `publish_replace_policy = REPLACE_ROLE`:

- overwrite `ik_main_key` (or whichever role)
    
- do not touch gallery unless role is a gallery role
    

---

# 9) Medusa & Keystone URL building (configuration + presets)

## 9.1 Shared helper

Create a shared function used by both apps:

- `base = IMAGEKIT_URL_ENDPOINT`
    
- `url = base + key`
    
- `tr = buildTransformString(contextPreset)`
    
- `final = url + "?tr=" + tr`
    

## 9.2 Example presets (you can hardcode)

- PLP thumbnail: `w-450,h-450,c-at_max`
    
- PDP main: `w-1400`
    
- Zoom: `w-3000`
    
- Social square: `w-1080,h-1080,c-maintain_ratio`
    
- Hero banner: `w-2400`
    

(You can later move these presets into config without touching Akeneo.)

---

# 10) What users must fill in (per product) vs what is automated

## Automated by publisher

- all `ik_*_key` attributes
    
- `ik_gallery_json`
    
- document keys
    

## Filled by PIM users (Akeneo)

- `alt_main` (localized)
    
- optional alt fields for the roles you care about
    
- product copy and all non-media attributes (your family distributions)
    

**Creator responsibilities remain inside ResourceSpace** (tagging + approval).  
**Merchandising/accessibility responsibilities remain inside Akeneo** (alt text + completeness).

---

## If you want, I can turn this into two “configuration sheets”

1. **ResourceSpace metadata schema**: field name, type, allowed values, required/conditional rules
    
2. **Akeneo attribute catalog**: attribute code, type, localizable/scopable, level placement per family, required yes/no
    

Say the locales/channels you’ll use (e.g., `en_GB`, `es_ES`, channels `ecommerce`, `wholesale`) and whether any channel needs different imagery, and I’ll finalize the scopable/localizable flags and required rules accordingly.