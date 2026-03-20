---
title: AEM Commerce and CIF Integration Framework
impact: MEDIUM
impactDescription: CIF provides the commerce integration layer for AEM — incorrect patterns cause slow catalog pages, stale product data, and broken checkout flows
tags: commerce, cif, graphql, adobe-commerce, magento, product-teaser, catalog, venia, cif-core-components
---

## AEM Commerce and CIF Integration Framework

The Commerce Integration Framework (CIF) connects AEM to commerce backends (Adobe Commerce/Magento, third-party) via GraphQL. CIF Core Components provide ready-to-use product display, catalog navigation, search, and cart functionality.

---

### 1. CIF Architecture

```
AEM Author/Publish
├── CIF Core Components (product teaser, list, search, cart, checkout)
├── CIF Connector (GraphQL client)
│   └── Connects to commerce backend via GraphQL
├── CIF URL Provider (commerce URL routing)
└── Product/Category Pickers (dialog integration)
        │
        ▼
Commerce Backend (Adobe Commerce / Magento / 3rd party)
├── GraphQL API (catalog, cart, checkout)
├── Product data, pricing, inventory
└── Order management
```

---

### 2. CIF Cloud Configuration

```json
// /conf/mysite/settings/cloudconfigs/commerce/.content.xml
<jcr:root xmlns:jcr="http://www.jcp.org/jcr/1.0" xmlns:sling="http://sling.apache.org/jcr/sling/1.0"
    jcr:primaryType="cq:Page">
    <jcr:content
        jcr:primaryType="cq:PageContent"
        jcr:title="Commerce Configuration"
        sling:resourceType="commerce/gui/components/configuration/page"
        magentoGraphqlEndpoint="https://my-commerce.example.com/graphql"
        magentoStore="default"
        enableUIDSupport="{Boolean}true"
        httpHeaders="[Authorization=Bearer {token}]"
        cq:graphqlClient="default"/>
</jcr:root>
```

**Apply to content tree:**

```
/content/mysite → jcr:content → cq:conf = /conf/mysite
```

---

### 3. CIF Core Components

#### Product Teaser

```html
<!-- Sling resource type: core/cif/components/commerce/productteaser/v2/productteaser -->
<sly data-sly-use.productTeaser="com.adobe.cq.commerce.core.components.models.productteaser.ProductTeaser">
    <div class="cmp-productteaser" data-cmp-is="productteaser"
         data-product-sku="${productTeaser.sku}">
        <a href="${productTeaser.url}" class="cmp-productteaser__link">
            <img class="cmp-productteaser__image"
                 src="${productTeaser.image}" alt="${productTeaser.name}"/>
            <div class="cmp-productteaser__details">
                <h2 class="cmp-productteaser__name">${productTeaser.name}</h2>
                <div class="cmp-productteaser__price">
                    <span class="cmp-productteaser__price--regular">
                        ${productTeaser.priceRange.regularPrice @ format='0.00'}
                    </span>
                    <sly data-sly-test="${productTeaser.priceRange.hasDiscount}">
                        <span class="cmp-productteaser__price--special">
                            ${productTeaser.priceRange.finalPrice @ format='0.00'}
                        </span>
                    </sly>
                </div>
            </div>
        </a>
        <button class="cmp-productteaser__cta" data-action="addToCart"
                data-sku="${productTeaser.sku}">
            Add to Cart
        </button>
    </div>
</sly>
```

#### Product List (Category Page)

```html
<sly data-sly-use.productList="com.adobe.cq.commerce.core.components.models.productlist.ProductList">
    <div class="cmp-productlist">
        <h1 class="cmp-productlist__title">${productList.title}</h1>

        <div class="cmp-productlist__items">
            <sly data-sly-list.product="${productList.products}">
                <div class="cmp-productlist__item">
                    <a href="${product.url}">
                        <img src="${product.imageUrl}" alt="${product.name}" loading="lazy"/>
                        <span class="cmp-productlist__item-name">${product.name}</span>
                        <span class="cmp-productlist__item-price">${product.price}</span>
                    </a>
                </div>
            </sly>
        </div>

        <!-- Pagination -->
        <sly data-sly-test="${productList.totalPages > 1}">
            <nav class="cmp-productlist__pagination">
                <sly data-sly-list.page="${productList.pages}">
                    <a href="${page.url}" class="${page.current ? 'active' : ''}">${page.number}</a>
                </sly>
            </nav>
        </sly>
    </div>
</sly>
```

#### Search Bar

```html
<sly data-sly-use.searchbar="com.adobe.cq.commerce.core.components.models.searchbar.SearchBar">
    <div class="cmp-searchbar" data-cmp-is="searchbar"
         data-search-result-page="${searchbar.searchResultsPage}">
        <form class="cmp-searchbar__form" action="${searchbar.searchResultsPage}">
            <input type="search" name="search_query"
                   class="cmp-searchbar__input"
                   placeholder="Search products..."
                   autocomplete="off"/>
            <button type="submit" class="cmp-searchbar__submit">Search</button>
        </form>
    </div>
</sly>
```

---

### 4. Product and Category Pickers in Dialogs

```xml
<!-- Product picker field in component dialog -->
<product
    jcr:primaryType="nt:unstructured"
    sling:resourceType="commerce/gui/components/common/cifproductfield"
    fieldLabel="Product"
    name="./product"
    multiple="{Boolean}false"
    emptyText="Select a product"/>

<!-- Category picker field -->
<category
    jcr:primaryType="nt:unstructured"
    sling:resourceType="commerce/gui/components/common/cifcategoryfield"
    fieldLabel="Category"
    name="./category"
    multiple="{Boolean}false"
    selectionId="uid"
    emptyText="Select a category"/>
```

---

### 5. CIF URL Provider and Routing

#### URL Structure

```
Product pages:  /content/mysite/en/products/product-page.html/{url_key}.html
Category pages: /content/mysite/en/products/category-page.html/{url_path}.html

Example:
  /content/mysite/en/products/product-page.html/blue-hoodie.html
  /content/mysite/en/products/category-page.html/men/tops.html
```

#### URL Provider Configuration

```json
// com.adobe.cq.commerce.core.components.internal.services.UrlProviderImpl.cfg.json
{
    "productUrlTemplate": "/content/mysite/en/products/product-page.html/{{url_key}}.html",
    "categoryUrlTemplate": "/content/mysite/en/products/category-page.html/{{url_path}}.html"
}
```

#### Sling Mapping for Clean URLs

```json
// /etc/map.publish/https/mysite.com/products
{
    "jcr:primaryType": "sling:Mapping",
    "sling:internalRedirect": "/content/mysite/en/products"
}
```

---

### 6. Product Data Enrichment

Combine commerce data with AEM content:

```java
// Extend product teaser with AEM-authored content
@Model(adaptables = SlingHttpServletRequest.class,
       adapters = ProductTeaser.class,
       resourceType = "myproject/components/commerce/productteaser")
public class CustomProductTeaser implements ProductTeaser {

    @Self
    @Via(type = ResourceSuperType.class)
    private ProductTeaser delegate; // CIF Core Component

    @ValueMapValue(name = "promotionBadge")
    @Default(values = "")
    private String promotionBadge; // AEM-authored field

    @ValueMapValue(name = "customCtaText")
    @Default(values = "Add to Cart")
    private String customCtaText;

    @Override
    public String getName() { return delegate.getName(); }

    @Override
    public String getSku() { return delegate.getSku(); }

    @Override
    public AbstractPriceRange getPriceRange() { return delegate.getPriceRange(); }

    @Override
    public String getUrl() { return delegate.getUrl(); }

    @Override
    public String getImage() { return delegate.getImage(); }

    // Custom methods
    public String getPromotionBadge() { return promotionBadge; }
    public String getCustomCtaText() { return customCtaText; }
}
```

---

### 7. Cart and Checkout Integration

#### Cart Component (Client-Side)

```javascript
// CIF uses Peregrine (React-based) for cart/checkout
// For headless, use the Commerce GraphQL directly:

async function addToCart(sku, quantity = 1) {
    const cartId = getCartId();
    const mutation = `
        mutation AddToCart($cartId: String!, $sku: String!, $qty: Float!) {
            addSimpleProductsToCart(input: {
                cart_id: $cartId
                cart_items: [{ data: { sku: $sku, quantity: $qty } }]
            }) {
                cart {
                    items { id product { name sku } quantity }
                    prices { grand_total { value currency } }
                }
            }
        }
    `;

    const response = await fetch('/api/graphql', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
            query: mutation,
            variables: { cartId, sku, qty: quantity }
        })
    });
    return response.json();
}
```

---

### 8. Performance: Catalog Caching

#### GraphQL Response Caching

```json
// com.adobe.cq.commerce.graphql.client.impl.GraphqlClientImpl~default.cfg.json
{
    "identifier": "default",
    "url": "https://my-commerce.example.com/graphql",
    "httpMethod": "GET",
    "cacheConfigurations": [
        "products:max-age=300",
        "categories:max-age=600",
        "cmsBlocks:max-age=3600"
    ]
}
```

#### Dispatcher Caching for Commerce Pages

```
# Cache product/category pages with short TTL
/0100 {
    /glob "*.html"
    /type "allow"
}

# Cache GraphQL responses at Dispatcher
# (only for persisted queries / GET requests)
/0200 {
    /glob "/api/graphql*"
    /type "allow"
}
```

---

### 9. Multi-Store and Multi-Currency

```json
// Store-specific cloud config
// /conf/mysite-emea/settings/cloudconfigs/commerce
{
    "magentoGraphqlEndpoint": "https://my-commerce.example.com/graphql",
    "magentoStore": "emea_en",
    "httpHeaders": ["Store=emea_en", "Content-Currency=EUR"]
}
```

```
Content tree:
/content/mysite
  ├── en/    → cq:conf=/conf/mysite-us    (store: us_en, currency: USD)
  ├── de/    → cq:conf=/conf/mysite-emea  (store: emea_de, currency: EUR)
  └── fr/    → cq:conf=/conf/mysite-emea  (store: emea_fr, currency: EUR)
```

---

### 10. Anti-Patterns

#### Direct REST Calls Instead of CIF GraphQL

```java
// WRONG — bypassing CIF, calling commerce REST directly
HttpClient client = HttpClientBuilder.create().build();
HttpGet get = new HttpGet("https://commerce.example.com/rest/V1/products/" + sku);

// CORRECT — use CIF GraphQL client (handles caching, auth, error handling)
@Reference
private GraphqlClient graphqlClient;
```

#### Storing Product Data in JCR

```
// WRONG — importing product catalog into AEM JCR
/content/products/sku-12345
  jcr:title = "Blue Hoodie"
  price = 49.99
// This data goes stale immediately

// CORRECT — fetch live from commerce backend via CIF GraphQL
// Product data lives in the commerce system, AEM enriches with content
```

#### Not Caching GraphQL Responses

```
// WRONG — every page render hits the commerce GraphQL endpoint
// Results: slow pages, commerce backend overload

// CORRECT — configure GraphQL client caching
// AND cache product pages at Dispatcher with appropriate TTL
```
