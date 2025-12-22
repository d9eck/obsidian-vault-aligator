Below is the **expanded Phase 2 – Data Modeling & Taxonomy Blueprint** for Akeneo PIM v7, designed specifically for your ceramic manufacturing business.  
This document is ready for Obsidian—fully markdown, clean structure, no dependencies.

---
# Phase 2 – Data Modeling & Taxonomy Blueprint

### Akeneo PIM v7 – Ceramics Manufacturing

---

## 1. Design Principles

Before configuring anything in Akeneo, the data model must follow principles that work **operationally** and **artistically**.

### 1.1 Core Principles

- **Expressive, not overwhelming:** Capture craftsmanship but keep enrichment manageable.    
- **Separation of concerns:**    
    - Technical data → responsibilities of Production        
    - Storytelling data → Marketing & Artisans        
- **Reuse + maintainability:**  
    Use **reference entities** (glazes, clay bodies, artisans) to avoid duplication.    
- **Scalable to collections:**  
    Collections and seasonal releases must be easy to manage.    
- **Variant ready:**  
    Support color/size/shape variations without data duplication.    
- **Channel-driven modeling:**  
    Only create attributes that serve a business need (e-commerce, retailer, print catalog).
    

---

## 2. Product Families – Final Structure

In **Akeneo PIM v7 CE**, a product belongs to **one Family** (one attribute set + completeness rules). That means your families should be **stable and operationally meaningful** (what attributes you must capture), not “marketing concepts” that cut across types.
Below is the recommended final family list, **optimized for ceramics**.

### 2.1 Families to Configure


Below is a pragmatic structure that balances **completeness**, **variant modeling**, and **team usability**:

#### 1) Art & One-of-a-kind Works
**Family:** `ART_UNIQUE`  
Best for: sculptures, wall pieces, one-off decorative objects  
#### 2) Tableware — Flatware
**Family:** `TABLEWARE_FLAT`  
Best for: plates, platters, trays  
#### 3) Tableware — Hollowware
**Family:** `TABLEWARE_HOLLOW`  
Best for: bowls (cereal/pasta/serving), ramekins  
#### 4) Drinkware
**Family:** `DRINKWARE`  
Best for: cups, mugs, tumblers  
#### 5) Vessels
**Family:** `VESSELS`  
Best for: vases, pitchers, jars, carafes  
#### 6) Tiles & Architectural Ceramics
**Family:** `TILES_DECOR`  
Best for: field tiles, relief tiles, panels  
#### 7) Sets & Bundles
**Family:** `SETS_BUNDLES`  
Best for: place settings, curated sets  



### 2.2 Family Variants

## Family: `ART_UNIQUE`

### Variant definition

**No variants (no Family Variant).**  
Reason: each piece is one sellable item; variants would create fake structure.

---

## Family: `TABLEWARE_FLAT`

### Family Variant

- **Code:** `FV_TW_FLAT_SIZE_GLAZE`
    
- **Levels:** 2
    
- **Level 1 axis:** `size_code` (Dinner / Salad / Dessert / Platter…)
    
- **Level 2 axis:** `glaze_primary` _(label as “Glaze / Colorway”)_
    

### Attribute distribution (Akeneo UI columns)

**COMMON (root product model)**

- `product_name`, `short_description`, `long_description`
    
- `product_form`
    
- `collection_line`, `profile_shape`
    
- `clay_body`
    
- `handmade_variation_note`
    
- Safety & care (constant across the line):
    
    - `food_contact_intended`, `food_safe_statement`, `lead_cadmium_compliance`
        
    - `dishwasher_safe`, `microwave_safe`
        
- Story/design:
    
    - `design_notes`
        
- Ops:
    
    - `country_of_origin`
        

**LEVEL 1 (by size_code) – sub product model**

- Axis: `size_code`
    
- Dimensions & physical:
    
    - `diameter`, `height`, `foot_diameter`, `rim_width`
        
- Use:
    
    - `stackable`
        
- Logistics/pricing (recommended here):
    
    - `item_weight`
        
    - `price` _(scopable per channel)_
        

**LEVEL 2 (by glaze/colorway) – variant products (sellable SKUs)**

- Axis: `glaze_primary` (“Glaze / Colorway”)
    
- Commercial identity:
    
    - `sku`
        
- Media (glaze-specific):
    
    - `image_main`, `image_topdown`, `image_side_profile`
        
- Appearance (if glaze drives it):
    
    - `finish` _(unless you keep finish constant for a whole line)_
        

**If you truly need price by glaze:** move `price` from Level 1 → Level 2.

---

## Family: `TABLEWARE_HOLLOW`

### Family Variant

- **Code:** `FV_TW_HOLLOW_SIZE_GLAZE`
    
- **Levels:** 2
    
- **Level 1 axis:** `size_code` (Ramekin / Cereal / Pasta / Serving…)
    
- **Level 2 axis:** `glaze_primary` (“Glaze / Colorway”)
    

### Attribute distribution

**COMMON**

- `product_name`, `short_description`, `long_description`
    
- `product_form`
    
- `collection_line`
    
- `clay_body`
    
- `handmade_variation_note`
    
- Safety & care:
    
    - `food_contact_intended`, `food_safe_statement`, `lead_cadmium_compliance`
        
    - `dishwasher_safe`, `microwave_safe`
        
- Story:
    
    - `serving_suggestions`
        
- Ops:
    
    - `country_of_origin`
        

**LEVEL 1 (by size_code)**

- Axis: `size_code`
    
- Dimensions/functional:
    
    - `diameter`, `height`, `depth`, `capacity`
        
- Use:
    
    - `stackable`, `nesting_note`
        
- Logistics/pricing:
    
    - `item_weight`, `price`
        

**LEVEL 2 (by glaze/colorway)**

- Axis: `glaze_primary`
    
- `sku`
    
- Media:
    
    - `image_main`, `image_in_hand` _(scale perception matters a lot for bowls)_
        
- Appearance:
    
    - `finish`
        

---

## Family: `DRINKWARE`

### Family Variant

- **Code:** `FV_DRINK_CAPACITY_GLAZE`
    
- **Levels:** 2
    
- **Level 1 axis:** `capacity_code` (Espresso / 200ml / 350ml / Mug…)
    
- **Level 2 axis:** `glaze_primary` (“Glaze / Colorway”)
    

### Attribute distribution

**COMMON**

- `product_name`, `short_description`, `long_description`
    
- `product_form`
    
- `collection_line`
    
- `clay_body`
    
- `handmade_variation_note`
    
- Safety & care:
    
    - `food_contact_intended`, `food_safe_statement`, `lead_cadmium_compliance`
        
    - `dishwasher_safe`, `microwave_safe`
        
- Ops:
    
    - `country_of_origin`
        

**LEVEL 1 (by capacity_code)**

- Axis: `capacity_code`
    
- Functional/ergonomics:
    
    - `capacity`, `hot_drink_suitable`
        
    - `handle_type`, `lip_profile`
        
    - `sip_feel_note`
        
- Dimensions/logistics:
    
    - `height`, `diameter`, `item_weight`
        
- Pricing:
    
    - `price`
        

**LEVEL 2 (by glaze/colorway)**

- Axis: `glaze_primary`
    
- `sku`
    
- Media:
    
    - `image_main`, `image_side_profile`
        
- Appearance:
    
    - `finish`
        

---

## Family: `VESSELS`

### Family Variant

- **Code:** `FV_VESSELS_SIZE_GLAZE`
    
- **Levels:** 2
    
- **Level 1 axis:** `size_code` _(or your height band options: Small/Medium/Large)_
    
- **Level 2 axis:** `glaze_primary` (“Glaze / Colorway”)
    

### Attribute distribution

**COMMON**

- `product_name`, `short_description`, `long_description`
    
- `product_form`
    
- `vessel_type`
    
- `clay_body`
    
- `handmade_variation_note`
    
- Story:
    
    - `use_suggestions`
        
- Ops:
    
    - `country_of_origin`
        

**LEVEL 1 (by size_code)**

- Axis: `size_code`
    
- Physical/functional:
    
    - `height`, `width`, `opening_diameter`, `capacity`
        
    - `watertight`
        
    - `has_handle`, `has_spout`
        
    - `food_contact_intended` _(pitchers/carafes: Yes; decor-only vases: No)_
        
    - `lead_cadmium_compliance` _(only if food-contact intended)_
        
- Care:
    
    - `dishwasher_safe` _(if applicable)_
        
- Logistics/pricing:
    
    - `item_weight`, `price`
        

**LEVEL 2 (by glaze/colorway)**

- Axis: `glaze_primary`
    
- `sku`
    
- Media:
    
    - `image_main`, `image_in_context`
        
- Appearance:
    
    - `finish`
        

---

## Family: `TILES_DECOR`

### Family Variant

- **Code:** `FV_TILE_FORMAT_GLAZE`
    
- **Levels:** 2
    
- **Level 1 axis:** `tile_shape_size` (e.g., 2x6, 4x4, Hex…)
    
- **Level 2 axis:** `glaze_primary` (“Glaze / Colorway”)
    

### Attribute distribution

**COMMON**

- `product_name`, `short_description`, `long_description`
    
- `tile_product_type`
    
- `clay_body`
    
- `handmade_variation_note` _(tile version)_
    
- Ops:
    
    - `country_of_origin`
        

**LEVEL 1 (by tile_shape_size)**

- Axis: `tile_shape_size`
    
- Geometry/spec:
    
    - `nominal_length`, `nominal_width`, `actual_length`, `actual_width`, `thickness`
        
    - `edge_type`
        
- Performance & suitability:
    
    - `water_absorption_pct`, `frost_resistant`, `application_area`
        
    - `dcof_wet` _(required when floor application is allowed)_
        
    - `shade_variation`
        
- Commercial/fulfillment:
    
    - `made_to_order`, `lead_time_weeks`
        
- Packaging:
    
    - `pieces_per_box`, `coverage_per_box_sqm`, `box_weight`
        
- Install:
    
    - `recommended_grout_joint_mm`
        
- Docs (usually format-level):
    
    - `file_spec_sheet`, `file_install_guide`
        
- Pricing/logistics (recommended here):
    
    - `price`, `item_weight` _(or keep box weight only; depends on your shipping model)_
        

**LEVEL 2 (by glaze/colorway)**

- Axis: `glaze_primary`
    
- `sku`
    
- Media (glaze-dependent):
    
    - `image_main`, `image_installed`
        
- Story/compliance notes:
    
    - `variation_note`
        
    - `prop65_warning` _(only if selling into CA/US and applicable)_
        

**If finish is independently selectable (Matte/Gloss for the same glaze option):**

- Add **a second Level 2 axis**: `finish`  
    This increases variant count, but it’s the cleanest way when finish is truly a selectable option.
    

---

## Family: `SETS_BUNDLES`


- **Code:** `FV_SET_GLAZE`
    
- **Levels:** 1
    
- **Level 1 axis:** `glaze_primary` (“Glaze / Colorway”)
    

**COMMON (root product model)**

- `product_name`, `short_description`, `long_description`
    
- `bundle_type`, `bundle_components_text`, `bundle_serves`
    
- `care_instructions`
    
- Safety:
    
    - `food_contact_intended`, `lead_cadmium_compliance`
        
- Ops:
    
    - `country_of_origin`
        

**LEVEL 1 (by glaze/colorway) – variant products**

- Axis: `glaze_primary`
    
- `sku`
    
- `price`
    
- `image_main`, `image_components_flatlay`
    
- `bundle_components` _(associations to component SKUs per glaze)_
    
- `item_weight`


---

## 4. Category Trees

Akeneo allows multiple trees. For ceramics, this is a huge strategic advantage.

### 4.1 Tree 1 – Sales / E-commerce Taxonomy (Customer-facing)

```
Ceramics
 ├── Tableware
 │    ├── Plates
 │    │    ├── Dinner Plates
 │    │    └── Side Plates
 │    ├── Bowls
 │    │    ├── Soup Bowls
 │    │    ├── Serving Bowls
 │    │    └── Ramen Bowls
 │    ├── Cups & Mugs
 │    └── Drinkware
 ├── Vessels
 │    ├── Vases
 │    ├── Jars
 │    └── Pitchers
 └── Art
      ├── Sculptures
      ├── Wall Art
      └── Limited Editions
```



---

## 5. Attribute Groups (Organizational Backbone)

Attribute groups keep Akeneo clean. Below is the ideal group set.

### 5.1 Attribute Group List

1. **Core Identification**    
2. **Dimensions**    
3. **Materials & Composition**    
4. **Craftsmanship & Process**    
5. **Functional Properties**    
6. **Aesthetic & Style**    
7. **Artisan & Origin**    
8. **Edition & Series Info**    
9. **Marketing & Storytelling**    
10. **SEO & Channel-specific**    
11. **Packaging & Logistics**    
12. **Regulatory**
13. **Assets / Media Links**

---

## 6. Attribute Schema (Complete List)

This is the heart of your PIM. Below is the definitive attribute catalog.

### 6.1 Core Identification

|Attribute|Type|Notes|
|---|---|---|
|`sku`|Identifier|Required|
|`internal_reference`|Text|From ERP or workshop|
|`name`|Text (localizable)|Display name|
|`product_model_code`|Text|For variants|

---

### 6.2 Dimensions

|Attribute|Type|Example|
|---|---|---|
|`height`|Measurement|in cm|
|`width`|Measurement||
|`depth`|Measurement||
|`diameter`|Measurement||
|`weight`|Measurement||
|`volume`|Measurement|ml (for vessels)|

---

### 6.3 Materials & Composition

|Attribute|Type|Notes|
|---|---|---|
|`clay_body`|Reference entity|Stoneware / Porcelain|
|`clay_color_raw`|Text|Color before firing|
|`clay_color_fired`|Text|Final color|
|`glaze`|Reference entity|Fully defined glaze object|
|`glaze_application_method`|Simple select|Dipped / Poured / Sprayed / Brushed|
|`number_of_firings`|Simple select|1 / 2 / 3+|
|`firing_temperature`|Measurement|Celsius|
|`firing_atmosphere`|Simple select|Electric / Gas reduction / Wood-fired / Raku|

---

### 6.4 Craftsmanship & Process

|Attribute|Type|
|---|---|
|`forming_technique`|Multi-select (wheel-thrown, hand-built, slab-built, cast)|
|`decorative_technique`|Multi-select (sgraffito, slip trailing, carving, painting)|
|`process_story`|Long text (localizable)|
|`making_duration`|Text (optional)|
|`special_process_notes`|Long text|

---

### 6.5 Functional Properties

|Attribute|Type|Example|
|---|---|---|
|`food_safe`|Boolean|Yes/no|
|`dishwasher_safe`|Boolean|Yes/no|
|`microwave_safe`|Boolean|Yes/no|
|`oven_safe`|Boolean|Yes/no|
|`care_instructions`|Long text|For customers|

---

### 6.6 Aesthetic & Style

|Attribute|Type|
|---|---|
|`color_palette`|Multi-select|
|`surface_finish`|Simple select (matte, satin, glossy, textured)|
|`design_motif`|Multi-select|
|`style_tags`|Multi-select|
|`collection`|Reference entity|

---

### 6.7 Artisan & Origin

|Attribute|Type|
|---|---|
|`artisan`|Reference entity|
|`studio_location`|Text|
|`year_made`|Number|
|`origin_story`|Long text|

---

### 6.8 Editions & Series (for limited editions)

|Attribute|Type|
|---|---|
|`edition_size`|Number|
|`edition_number`|Number|
|`series_name`|Text or reference entity|

---

### 6.9 Marketing & Storytelling

|Attribute|Type|
|---|---|
|`short_description`|Text|
|`long_description`|Long text|
|`craft_story`|Long text|
|`inspiration`|Long text|
|`unique_features`|Multi-line text|
|`marketing_highlights`|Multi-select|
|`keywords`|Text|

---

### 6.10 SEO & Channel-Specific Fields

|Attribute|Type|
|---|---|
|`seo_title`|Text|
|`seo_description`|Text|
|`url_slug`|Text|

These are **localizable + scopable** (per channel).

---

### 6.11 Packaging & Logistics

|Attribute|Type|
|---|---|
|`packaging_type`|Simple select|
|`packaging_dimensions`|Measurement (multiple fields)|
|`packaging_weight`|Measurement|
|`country_of_origin`|Simple select|

---

### 6.12 Regulatory / Certificates

|Attribute|Type|
|---|---|
|`certifications`|Multi-select|
|`compliance_notes`|Long text|
|`regulatory_documents`|Asset collection|

---

## 7. Reference Entities – Detailed Models

### 7.1 Glaze (Reference Entity)

|Field|Type|
|---|---|
|`name`|Text|
|`color_palette`|Multi-select|
|`finish`|Simple select|
|`texture`|Simple select|
|`opacity`|Simple select (transparent, semi-opaque, opaque)|
|`firing_temperature`|Measurement|
|`crazing_notes`|Text|
|`swatch_image`|Asset|
|`story`|Long text|

### 7.2 Clay Body (Reference Entity)

|Field|Type|
|---|---|
|`name`|Text|
|`composition`|Text|
|`firing_range`|Measurement|
|`shrinkage_rate`|Number|
|`color_after_firing`|Text|

### 7.3 Artisan (Reference Entity)

|Field|Type|
|---|---|
|`full_name`|Text|
|`bio`|Long text|
|`signature_style`|Text|
|`specialties`|Multi-select|
|`portrait_image`|Asset|
|`website`|URL|

### 7.4 Collection (Reference Entity)

|Field|Type|
|---|---|
|`name`|Text|
|`story`|Long text|
|`year_introduced`|Number|
|`main_colors`|Multi-select|
|`moodboard_images`|Asset collection|

---

## 8. Completeness Rules (by Family & Channel)

### 8.1 Example – Functional Tableware (e-commerce channel)

|Required Attribute|Category|
|---|---|
|name|Core|
|short_description|Marketing|
|long_description|Marketing|
|height/diameter|Dimensions|
|glaze|Materials|
|clay_body|Materials|
|forming_technique|Process|
|artisan|Origin|
|hero_image|Assets|
|care_instructions|Functional|
|seo_title / seo_description|SEO|

### 8.2 Example – Art Piece (Unique) (e-commerce channel)

Add these additional required fields:

- **origin_story**    
- **process_story**    
- **collection** (optional but recommended)    
- **edition info** (if editioned)    
- **gallery-ready images**
    

---

## 9. Rules Engine (Automation Ideas)

Sample rules that can be implemented in Akeneo’s rule engine:

- **Auto-assign categories** based on attributes  
    If `forming_technique = wheel-thrown` AND `diameter > 20 cm` AND `height < 5 cm` → Category: Plates.
    
- **Auto-link glaze swatches** to product assets  
    If `glaze = X`, link glaze swatch asset to asset collection.
    
- **Auto-generate SEO title**  
    `seo_title = name + " – Handmade Ceramic " + family_label`
    
- **Auto-assign collection styles**  
    If product belongs to “Ocean Mist”, assign style tags: Coastal, Organic, Blue Palette.
    

---

## 10. Required Akeneo Configuration Steps Checklist

### 10.1 Core Setup

-  Create families (5 total)    
-  Create variant families (as needed)    
-  Configure 3 category trees    
-  Add all 12 attribute groups    
-  Add all attributes with correct types/localizable/scope settings    
-  Create 4 reference entity types    
-  Create completeness rules for each family + channel    
-  Prepare data import templates    
-  Configure rules engine automations (Phase 2.5 optional)
    

---

# NEXT STEPS (Choose One)

I can now generate:

### **A. Full Attribute Name List (Akeneo-ready naming conventions)**

CamelCase / snake_case / readable names.

### **B. CSV Import Templates (for Families, Attributes, Reference Entities)**

Ready to load into Akeneo.

### **C. Product Model / Variant Configuration Examples**

Including real sample products for your catalog.

### **D. Enrichment workflow diagrams** (Markdown or Mermaid).

### **E. Image & Asset workflows** (naming conventions, folder logic, AI-assisted image metadata extraction).

Tell me which direction you'd like to go next.