  

## 8. Catalog Sync / Anti-Corruption Layer

  

The Catalog Sync is a logical component. It MAY be implemented as a standalone service, worker, serverless function, or Medusa-adjacent integration module. Regardless of runtime form, its responsibilities remain the same.

  

### 8.1 Responsibilities

  

The Catalog Sync MUST:

  

1. receive publish, update, unpublish, and archive events from Directus

2. fetch the canonical record from Directus

3. validate publishability and completeness

4. transform the Directus authoring model into the Medusa commerce projection

5. perform idempotent upserts into Medusa

6. record sync metadata for traceability

7. trigger storefront cache invalidation / revalidation for affected routes

  

### 8.2 Mapping rules

  

| Directus concept | Medusa projection |

|---|---|

| Product family | Product |

| Variant | Product variant |

| Taxonomy / collections | Collections or categories where needed |

| Base prices | Variant price entries / price lists |

| Primary asset references | Product or variant image references |

| Market publication scope | Region or sales-channel availability |

| Rich editorial blocks | Not projected; read directly from Directus by Storefront |

  

Only the subset of data required for sellability and commerce operations SHOULD be projected into Medusa.

  

---

  



---