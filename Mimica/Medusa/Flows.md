## 9. Key runtime flows

  

### 9.1 Product publish flow

  

1. A merchandiser or content editor publishes a product or content item in Directus.

2. Directus emits a versioned catalog/content event.

3. Catalog Sync receives the event and fetches the canonical entity from Directus.

4. Catalog Sync validates completeness and market eligibility.

5. Catalog Sync transforms and upserts the Medusa commerce projection.

6. Storefront revalidation is triggered for affected product, category, collection, landing, or navigation routes.

7. Observability events are emitted for success, failure, and latency.

  

### 9.2 Product page request flow

  

1. The shopper requests a route.

2. The storefront resolves active market, locale, and currency.

3. The storefront fetches Directus content for the route.

4. The storefront fetches Medusa commerce data for the product or listing.

5. The storefront composes a view model and renders the page.

6. Interactive purchase actions call Medusa through approved storefront server-side boundaries.

  

### 9.3 Checkout flow

  

1. The shopper adds a variant to cart.

2. Medusa validates variant availability, market eligibility, price selection, and promotions.

3. The storefront retrieves shipping and payment options from Medusa.

4. Medusa coordinates payment authorization and order creation.

5. Medusa manages fulfillment and post-order operational state.

6. Directus is not part of the checkout critical path.

  