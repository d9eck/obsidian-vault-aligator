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