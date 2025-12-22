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

### 1) Art & One-of-a-kind Works

**Family:** `ART_UNIQUE`  
Best for: sculptures, wall pieces, one-off decorative objects  
Why: you’ll need richer storytelling + provenance fields (and less variant logic).

**Must-have attribute ideas (ceramics-aware):**

- Technique (handbuilt / thrown / slab / sculpted)
    
- Clay body (stoneware/porcelain/earthenware + notes)
    
- Firing (oxidation/reduction; cone range)
    
- Glaze name(s), surface finish, special effects (crystalline, ash, crawling, etc.)
    
- Signature/mark, year made
    
- **Uniqueness** statement (what makes this piece singular)
    

### 2) Tableware — Flatware

**Family:** `TABLEWARE_FLAT`  
Best for: plates, platters, trays  
Why: diameter/foot/rim/stacking attributes are essential and consistent.

### 3) Tableware — Hollowware

**Family:** `TABLEWARE_HOLLOW`  
Best for: bowls (cereal/pasta/serving), ramekins  
Why: **capacity + depth profile** matters a lot and should be required.

### 4) Drinkware

**Family:** `DRINKWARE`  
Best for: cups, mugs, tumblers  
Why: handle type, capacity, lip feel notes, thermal comfort become required.

### 5) Vessels

**Family:** `VESSELS`  
Best for: vases, pitchers, jars, carafes  
Why: you’ll want **watertightness**, spout/handle presence, opening diameter, etc.

### 6) Tiles & Architectural Ceramics

**Family:** `TILES_DECOR`  
Best for: field tiles, relief tiles, panels  
Why: tile needs a different technical schema (size/thickness/edge/installation constraints). Heath/Fireclay-style configurability is a different universe than a mug. [Heath Ceramics+2Fireclay Tile+2](https://www.heathceramics.com/collections/vases-objects)

### 7) Sets & Bundles

**Family:** `SETS_BUNDLES`  
Best for: place settings, curated sets  
Why: sets behave like their own commercial object (components, packaging, bundle logic). This is prominent in tableware leaders.

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