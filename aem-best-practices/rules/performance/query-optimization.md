---
title: JCR/Oak Query Optimization for AEM Cloud Service
impact: HIGH
impactDescription: Unoptimized queries cause traversal warnings, slow page rendering, and can trigger Oak query limits that break content lists entirely
tags: querybuilder, jcr-sql2, xpath, oak-index, lucene, traversal, pagination, content-fragments, dam, sling-models, performance
---

## JCR/Oak Query Optimization

Efficient repository queries are critical for frontend-visible performance. Slow queries directly impact page load times for content lists, tag-based navigation, search results, and related content components.

---

### 1. QueryBuilder API: Predicates, Pagination, and Result Control

QueryBuilder is the recommended high-level API for most AEM queries. It converts predicate maps to XPath internally and provides pagination, facets, and result extraction.

**Correct -- paginated content list with guessTotal:**

```java
// Sling Model for a "Latest Articles" component
@Model(adaptables = SlingHttpServletRequest.class)
public class LatestArticles {

    @Self
    private SlingHttpServletRequest request;

    @OSGiService
    private QueryBuilder queryBuilder;

    @ValueMapValue(name = "articleRoot", injectionStrategy = InjectionStrategy.OPTIONAL)
    private String articleRoot;

    @ValueMapValue(name = "limit", injectionStrategy = InjectionStrategy.OPTIONAL)
    private int limit = 10;

    public List<Article> getArticles() {
        Map<String, String> params = new HashMap<>();
        params.put("path", articleRoot != null ? articleRoot : "/content/mysite/articles");
        params.put("type", "cq:Page");
        params.put("property", "jcr:content/cq:template");
        params.put("property.value", "/conf/mysite/settings/wcm/templates/article");
        params.put("orderby", "@jcr:content/cq:lastModified");
        params.put("orderby.sort", "desc");
        params.put("p.limit", String.valueOf(limit));
        params.put("p.guessTotal", "100");  // Avoids full count traversal

        Session session = request.getResourceResolver().adaptTo(Session.class);
        Query query = queryBuilder.createQuery(PredicateGroup.create(params), session);
        SearchResult result = query.getResult();

        return result.getHits().stream()
            .map(this::toArticle)
            .collect(Collectors.toList());
    }
}
```

**REST API equivalent:**

```
/bin/querybuilder.json?path=/content/mysite/articles
    &type=cq:Page
    &property=jcr:content/cq:template
    &property.value=/conf/mysite/settings/wcm/templates/article
    &orderby=@jcr:content/cq:lastModified
    &orderby.sort=desc
    &p.limit=10
    &p.guessTotal=100
```

**Incorrect -- unbounded query without guessTotal:**

```java
// Fetches ALL results, iterates entire result set for count
params.put("p.limit", "-1");
// Missing p.guessTotal -- forces full traversal for total count
```

#### Key Predicates Reference

| Predicate | Usage | Example |
|-----------|-------|---------|
| `path` | Repository subtree | `path=/content/mysite` |
| `type` | Node type filter | `type=cq:Page` |
| `property` | Property match | `property=jcr:content/cq:tags` |
| `property.value` | Property value | `property.value=mysite:topic/tech` |
| `property.operation` | Comparison | `property.operation=like` |
| `fulltext` | Full-text search | `fulltext=cloud migration` |
| `tagid` | Tag-based filter | `tagid=mysite:topic/tech` |
| `orderby` | Sort field | `orderby=@jcr:content/jcr:lastModified` |
| `orderby.sort` | Sort direction | `orderby.sort=desc` |
| `daterange.property` | Date range filter | `daterange.property=jcr:content/cq:lastModified` |
| `daterange.lowerBound` | Date lower bound | `daterange.lowerBound=2024-01-01` |
| `relativedaterange.lowerBound` | Relative date | `relativedaterange.lowerBound=-30d` |
| `nodename` | Node name pattern | `nodename=article*` |
| `group.p.or` | OR grouping | `group.p.or=true` |
| `p.limit` | Page size | `p.limit=20` |
| `p.offset` | Pagination offset | `p.offset=20` |
| `p.guessTotal` | Estimated total | `p.guessTotal=100` |
| `p.hits` | Result detail level | `p.hits=selective&p.properties=jcr:path` |

#### Multiple Predicates of Same Type

Use numeric prefixes to combine predicates:

```
1_property=jcr:content/cq:tags
1_property.value=mysite:topic/tech
2_property=jcr:content/pageType
2_property.value=article
```

#### OR Logic Within Groups

```
group.p.or=true
group.1_property=jcr:content/cq:tags
group.1_property.value=mysite:topic/tech
group.2_property=jcr:content/cq:tags
group.2_property.value=mysite:topic/cloud
```

---

### 2. QueryBuilder vs JCR-SQL2 vs XPath

| Feature | QueryBuilder | JCR-SQL2 | XPath |
|---------|-------------|----------|-------|
| Abstraction level | High (predicate maps) | Medium (SQL-like) | Low (path expressions) |
| REST API | Yes (`/bin/querybuilder.json`) | No | No |
| Facet extraction | Built-in | Manual | Manual |
| Pagination | `p.limit`/`p.offset` | Manual | Manual |
| OR/AND logic | Group predicates | Standard SQL | Union paths |
| Performance | Converts to XPath internally | Direct Oak execution | Direct Oak execution |
| Joins | Not supported | `JOIN` syntax | Not supported |
| Best for | Component queries, REST | Complex joins, aggregations | Simple path queries |

**When to use each:**

- **QueryBuilder**: Default choice for component-driven content lists, tag queries, paginated results. Easiest to maintain and test.
- **JCR-SQL2**: When you need `JOIN` across node types or complex aggregations that QueryBuilder cannot express.
- **XPath**: Rarely needed directly. QueryBuilder generates XPath internally. Use only for very simple path-based lookups.

**JCR-SQL2 example (join pattern):**

```java
String sql2 = "SELECT page.* FROM [cq:Page] AS page "
    + "INNER JOIN [nt:unstructured] AS content ON ISCHILDNODE(content, page) "
    + "WHERE ISDESCENDANTNODE(page, '/content/mysite') "
    + "AND content.[cq:template] = '/conf/mysite/settings/wcm/templates/article' "
    + "AND content.[cq:lastModified] > CAST('2024-01-01T00:00:00.000Z' AS DATE) "
    + "ORDER BY content.[cq:lastModified] DESC";

QueryManager qm = session.getWorkspace().getQueryManager();
javax.jcr.query.Query query = qm.createQuery(sql2, Query.JCR_SQL2);
query.setLimit(20);
QueryResult result = query.execute();
```

---

### 3. Oak Index Types

AEM Cloud Service only supports **Lucene indexes**. Property indexes and ordered indexes from AEM 6.x are not available.

#### Lucene Index Capabilities

- Full-text search (analyzed/tokenized properties)
- Property value matching (exact, like, range)
- Path restrictions (`evaluatePathRestrictions`)
- Ordering on indexed properties
- Facet extraction
- Aggregate indexing (parent/child property consolidation)

#### Out-of-the-Box Indexes

| Index | Purpose |
|-------|---------|
| `cqPageLucene` | Page queries (cq:Page) |
| `damAssetLucene` | Asset queries (dam:Asset) |
| `lucene` | Generic fallback (being removed) |

Always extend OOTB indexes rather than creating new ones for the same node types.

#### Custom Index Definition Structure

Index definitions live in the project at `ui.apps/src/main/content/jcr_root/_oak_index/`:

```xml
<!-- ui.apps/src/main/content/jcr_root/_oak_index/acme.articleIndex-1-custom-1/.content.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<jcr:root xmlns:jcr="http://www.jcp.org/jcr/1.0"
          xmlns:nt="http://www.jcp.org/jcr/nt/1.0"
          xmlns:oak="http://jackrabbit.apache.org/oak/ns/1.0"
    jcr:primaryType="oak:QueryIndexDefinition"
    type="lucene"
    async="[async,nrt]"
    compatVersion="{Long}2"
    evaluatePathRestrictions="{Boolean}true"
    includedPaths="[/content/mysite]"
    queryPaths="[/content/mysite]"
    tags="[articleQuery]"
    selectionPolicy="tag">
    <indexRules jcr:primaryType="nt:unstructured">
        <cq:Page jcr:primaryType="nt:unstructured">
            <properties jcr:primaryType="nt:unstructured">
                <template
                    jcr:primaryType="nt:unstructured"
                    name="jcr:content/cq:template"
                    propertyIndex="{Boolean}true"/>
                <lastModified
                    jcr:primaryType="nt:unstructured"
                    name="jcr:content/cq:lastModified"
                    propertyIndex="{Boolean}true"
                    ordered="{Boolean}true"
                    type="Date"/>
                <tags
                    jcr:primaryType="nt:unstructured"
                    name="jcr:content/cq:tags"
                    propertyIndex="{Boolean}true"/>
            </properties>
        </cq:Page>
    </indexRules>
    <tika jcr:primaryType="nt:folder">
        <config.xml/>
    </tika>
</jcr:root>
```

**Naming conventions:**

| Type | Pattern | Example |
|------|---------|---------|
| OOTB customization | `<name>-<ver>-custom-<n>` | `damAssetLucene-8-custom-1` |
| Fully custom | `<prefix>.<name>-<ver>-custom-<n>` | `acme.articleIndex-1-custom-1` |

**Required project configuration:**

```xml
<!-- ui.apps/pom.xml -->
<plugin>
    <groupId>org.apache.jackrabbit</groupId>
    <artifactId>filevault-package-maven-plugin</artifactId>
    <configuration>
        <allowIndexDefinitions>true</allowIndexDefinitions>
    </configuration>
</plugin>
```

Add filter entry in `ui.apps/src/main/content/META-INF/vault/filter.xml`:

```xml
<filter root="/oak:index/acme.articleIndex-1-custom-1"/>
```

---

### 4. Detecting Slow Queries

#### Cloud Service: Developer Console

CRXDE query tool is not available in Cloud Service. Use the **Developer Console** (accessible via Cloud Manager) to:
- View query performance metrics
- Run explain plans
- Identify traversal warnings

#### Slow Query Threshold

Queries scanning more than **5,000 nodes** are flagged as slow. The Read Optimization Score should be 90% or above (ratio of matching results to scanned nodes).

#### Explain Plan Analysis

```java
// Programmatic explain plan
Map<String, String> params = new HashMap<>();
params.put("path", "/content/mysite");
params.put("type", "cq:Page");
params.put("property", "jcr:content/cq:tags");
params.put("property.value", "mysite:topic/tech");

Query query = queryBuilder.createQuery(PredicateGroup.create(params), session);
// Check the explain output before executing
String plan = query.getResult().getQueryStatement();
```

#### Warning Types

| Warning | Meaning | Action |
|---------|---------|--------|
| `Traversed X nodes` | No index used at all | Create or fix index |
| `Index-Traversed X nodes` | Index used but over-scanning | Add missing property to index |
| `Filter(...)` in plan | Post-index filtering | Ensure all predicates are indexed |

#### JMX Monitoring (AEM 6.x / SDK)

Access `/system/console/jmx` and check:
- `org.apache.jackrabbit.oak:QueryStat` -- slow query log
- `org.apache.jackrabbit.oak:QueryEngineSettings` -- traversal limits

---

### 5. Common Anti-Patterns

#### Traversal Queries (No Index)

**Incorrect:**

```java
// Queries on unindexed property -- causes full repository traversal
params.put("path", "/content");
params.put("property", "jcr:content/myCustomProperty");
params.put("property.value", "someValue");
// Oak will traverse ALL nodes under /content looking for matches
```

**Correct:**

```java
// Same query but with a custom index that includes myCustomProperty
// AND use tag-based index selection for predictability
params.put("path", "/content/mysite");  // Narrowest possible path
params.put("property", "jcr:content/myCustomProperty");
params.put("property.value", "someValue");
params.put("p.indexTag", "myCustomQuery");  // Ensures correct index
```

#### Unbounded Result Sets

**Incorrect:**

```java
params.put("p.limit", "-1");  // Returns ALL matching nodes
// If 50,000 pages match, all are loaded into memory
```

**Correct:**

```java
params.put("p.limit", "20");
params.put("p.guessTotal", "200");
// Frontend implements pagination or "Load More" pattern
```

#### Missing orderby Index Support

**Incorrect:**

```java
params.put("orderby", "@jcr:content/customDate");
params.put("orderby.sort", "desc");
// If customDate is not indexed with ordered=true, Oak sorts in memory
// This reads ALL matching nodes just to sort them
```

**Correct:**

```xml
<!-- In the index definition, add ordered=true for the sort property -->
<customDate
    jcr:primaryType="nt:unstructured"
    name="jcr:content/customDate"
    propertyIndex="{Boolean}true"
    ordered="{Boolean}true"
    type="Date"/>
```

#### N+1 Query Pattern in Sling Models

**Incorrect:**

```java
@Model(adaptables = Resource.class)
public class ArticleList {
    // First query: get all article pages
    public List<ArticleCard> getArticles() {
        // ... QueryBuilder returns 20 article pages
        return pages.stream().map(page -> {
            // N additional queries: each card queries for related tags, author, etc.
            Resource author = resolver.getResource(page.getAuthorPath());
            List<Tag> tags = tagManager.getTags(page.getResource());
            // Each iteration triggers separate JCR reads
            return new ArticleCard(page, author, tags);
        }).collect(Collectors.toList());
    }
}
```

**Correct:**

```java
@Model(adaptables = Resource.class)
public class ArticleList {
    public List<ArticleCard> getArticles() {
        Map<String, String> params = new HashMap<>();
        params.put("path", articleRoot);
        params.put("type", "cq:Page");
        params.put("p.limit", "20");
        params.put("p.guessTotal", "100");
        params.put("p.hits", "full");
        params.put("p.nodedepth", "3");  // Pre-fetch child nodes (jcr:content + children)

        // Single query returns pages with deep properties already loaded
        Query query = queryBuilder.createQuery(PredicateGroup.create(params), session);
        SearchResult result = query.getResult();

        return result.getHits().stream()
            .map(hit -> {
                Resource pageRes = hit.getResource();
                // Properties already loaded via p.nodedepth -- no additional queries
                ValueMap props = pageRes.getChild("jcr:content").getValueMap();
                String authorName = props.get("authorName", String.class);
                String[] tagIds = props.get("cq:tags", String[].class);
                return new ArticleCard(pageRes, authorName, tagIds);
            })
            .collect(Collectors.toList());
    }
}
```

---

### 6. Content Fragment and DAM Query Optimization

#### Content Fragment Queries

**Correct -- querying Content Fragments by model:**

```java
params.put("path", "/content/dam/mysite/articles");
params.put("type", "dam:Asset");
params.put("property", "jcr:content/data/cq:model");
params.put("property.value", "/conf/mysite/settings/dam/cfm/models/article");
params.put("orderby", "@jcr:content/jcr:lastModified");
params.put("orderby.sort", "desc");
params.put("p.limit", "10");
params.put("p.guessTotal", "100");
```

**For GraphQL-based access**, always use **persisted queries** instead of ad-hoc queries:

```
# Persisted query (CDN-cacheable, GET request)
GET /graphql/execute.json/mysite/articles;limit=10;offset=0

# NOT ad-hoc POST queries (not CDN-cacheable)
```

#### DAM Asset Queries

**Incorrect -- broad DAM search:**

```java
params.put("path", "/content/dam");  // Searches entire DAM tree
params.put("type", "dam:Asset");
params.put("property", "jcr:content/metadata/dc:format");
params.put("property.value", "image/jpeg");
// Potentially millions of assets to scan
```

**Correct -- scoped DAM search:**

```java
params.put("path", "/content/dam/mysite/campaign-2024");  // Narrow scope
params.put("type", "dam:Asset");
params.put("1_property", "jcr:content/metadata/dc:format");
params.put("1_property.value", "image/jpeg");
params.put("2_property", "jcr:content/metadata/dam:status");
params.put("2_property.value", "approved");
params.put("p.limit", "50");
params.put("p.guessTotal", "200");
// Extend damAssetLucene index if filtering on custom metadata
```

---

### 7. Lazy Loading Content Lists

#### Pagination Pattern (Offset-Based)

```java
// Servlet for AJAX pagination
@Component(service = Servlet.class, property = {
    "sling.servlet.paths=/bin/mysite/articles",
    "sling.servlet.methods=GET"
})
public class ArticleListServlet extends SlingSafeMethodsServlet {
    @Reference
    private QueryBuilder queryBuilder;

    @Override
    protected void doGet(SlingHttpServletRequest request, SlingHttpServletResponse response)
            throws IOException {
        int offset = Integer.parseInt(request.getParameter("offset"));
        int limit = Integer.parseInt(request.getParameter("limit"));

        Map<String, String> params = new HashMap<>();
        params.put("path", "/content/mysite/articles");
        params.put("type", "cq:Page");
        params.put("orderby", "@jcr:content/cq:lastModified");
        params.put("orderby.sort", "desc");
        params.put("p.limit", String.valueOf(limit));
        params.put("p.offset", String.valueOf(offset));
        params.put("p.guessTotal", "200");

        Session session = request.getResourceResolver().adaptTo(Session.class);
        SearchResult result = queryBuilder.createQuery(
            PredicateGroup.create(params), session).getResult();

        // Return JSON for frontend consumption
        JsonObject json = new JsonObject();
        json.addProperty("total", result.getTotalMatches());
        json.addProperty("hasMore", result.hasMore());
        // ... serialize hits
        response.setContentType("application/json");
        response.getWriter().write(json.toString());
    }
}
```

**Important**: For large offsets (>1000), offset-based pagination degrades because Oak must traverse all preceding results. Use **keyset pagination** instead:

```java
// Keyset pagination: use last item's sort value as cursor
params.put("daterange.property", "jcr:content/cq:lastModified");
params.put("daterange.upperBound", lastItemDate);  // Cursor from previous page
params.put("daterange.upperOperation", "<");
params.put("p.limit", "20");
// No p.offset needed -- starts from the cursor position
```

#### Frontend "Load More" Pattern

```javascript
// Frontend JS for load-more button
let offset = 0;
const limit = 10;

document.querySelector('.load-more').addEventListener('click', async () => {
    offset += limit;
    const response = await fetch(
        `/bin/mysite/articles?offset=${offset}&limit=${limit}`
    );
    const data = await response.json();
    renderArticles(data.results);

    if (!data.hasMore) {
        document.querySelector('.load-more').style.display = 'none';
    }
});
```

---

### 8. Query Result Caching

#### Resource Resolver Level

Sling's Resource Resolver caches resource reads within a single request. Accessing the same path multiple times in one request does not trigger multiple JCR reads.

#### Sling Model Caching

```java
@Model(adaptables = SlingHttpServletRequest.class,
       cache = true)  // Caches model instance per request
public class NavigationModel {
    private List<NavItem> items;

    @PostConstruct
    protected void init() {
        // Query executes once per request, even if model is adapted multiple times
        items = executeQuery();
    }
}
```

#### Application-Level Caching for Expensive Queries

```java
@Component(service = ArticleCacheService.class)
public class ArticleCacheService {
    // Use a simple TTL cache for frequently-accessed query results
    private final Cache<String, List<Article>> cache = CacheBuilder.newBuilder()
        .maximumSize(100)
        .expireAfterWrite(5, TimeUnit.MINUTES)
        .build();

    public List<Article> getTopArticles(ResourceResolver resolver) {
        return cache.get("topArticles", () -> executeQuery(resolver));
    }
}
```

**Warning**: Never cache ResourceResolver or Session objects -- they are request-scoped and will cause memory leaks or stale data.

---

### 9. Cloud Service Specifics

| Feature | AEM 6.x | AEM Cloud Service |
|---------|---------|-------------------|
| Index types | Property, Ordered, Lucene | **Lucene only** |
| CRXDE query tool | Available | **Not available** |
| Query debugging | `/libs/cq/search/content/querydebug.html` | **Developer Console** |
| Index deployment | Manual or content package | **Cloud Manager CI/CD pipeline** |
| Reindexing | Manual trigger | **Automatic during deployment** |
| Generic Lucene index | Available | **Being removed** (define specific indexes) |
| Traversal limit | Configurable | **Fixed at 100,000 nodes** |

**Developer Console query tools:**
- Query Performance Tool: view executed queries with performance metrics
- Explain Plan: analyze index usage for specific queries
- Slow Query Log: queries scanning >5,000 nodes

---

### 10. Practical Examples

#### Tag-Based Content List

```java
// "Related Articles" component -- finds pages sharing tags with current page
Map<String, String> params = new HashMap<>();
params.put("path", "/content/mysite");
params.put("type", "cq:Page");
params.put("tagid", currentPageTag);
params.put("tagid.property", "jcr:content/cq:tags");
params.put("excludepaths", currentPage.getPath());  // Exclude current page
params.put("orderby", "@jcr:content/cq:lastModified");
params.put("orderby.sort", "desc");
params.put("p.limit", "5");
params.put("p.guessTotal", "50");
```

#### Child Page Query (Navigation/Listing)

```java
// List child pages of a given root -- no QueryBuilder needed for direct children
Resource root = resolver.getResource("/content/mysite/articles");
Iterator<Page> children = root.adaptTo(Page.class).listChildren(
    new PageFilter(false, false),  // Skip hidden, invalid pages
    false  // Non-recursive (direct children only)
);
// For direct children, Page.listChildren() is faster than QueryBuilder
```

**When to use QueryBuilder for child listings:**

```java
// When you need filtering, sorting, or deep descendants
params.put("path", "/content/mysite/articles");
params.put("path.flat", "true");  // Direct children only (no deep descendants)
params.put("type", "cq:Page");
params.put("property", "jcr:content/articleCategory");
params.put("property.value", "technology");
params.put("orderby", "@jcr:content/publishDate");
params.put("orderby.sort", "desc");
params.put("p.limit", "10");
```

#### Search Component with Multiple Filters

```java
Map<String, String> params = new HashMap<>();
params.put("path", "/content/mysite");
params.put("type", "cq:Page");
params.put("fulltext", searchTerm);
params.put("fulltext.relPath", "jcr:content");

// Category filter (OR logic)
if (categories != null && categories.length > 0) {
    params.put("group.p.or", "true");
    for (int i = 0; i < categories.length; i++) {
        params.put("group." + (i + 1) + "_property", "jcr:content/articleCategory");
        params.put("group." + (i + 1) + "_property.value", categories[i]);
    }
}

// Date range filter
if (fromDate != null) {
    params.put("daterange.property", "jcr:content/cq:lastModified");
    params.put("daterange.lowerBound", fromDate);
}

params.put("orderby", "@jcr:score");
params.put("orderby.sort", "desc");
params.put("p.limit", "20");
params.put("p.offset", String.valueOf(page * 20));
params.put("p.guessTotal", "1000");
params.put("p.excerpt", "true");  // Include search excerpts
```

---

### Pitfalls

- Using `p.limit=-1` to fetch all results instead of paginating
- Omitting `p.guessTotal` on queries that may match thousands of nodes
- Querying `/content` or `/content/dam` without narrowing the path scope
- Sorting on properties not marked `ordered=true` in the index definition
- Creating new indexes for `dam:Asset` instead of extending `damAssetLucene`
- Using POST-based GraphQL queries instead of persisted queries (not CDN-cacheable)
- Caching `ResourceResolver` or `Session` objects across requests
- Not using `p.indexTag` when multiple custom indexes could match the same query
- Running queries in component HTL via `queryBuilder` use-object (use Sling Models instead)
- Large offset pagination (>1000) without switching to keyset pagination
