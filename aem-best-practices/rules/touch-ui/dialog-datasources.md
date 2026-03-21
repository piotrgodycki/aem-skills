---
title: Dynamic Dialog Data with Granite DataSources
impact: HIGH
impactDescription: DataSources power dynamic select options, autocomplete suggestions, and context-aware field population — essential for scalable dialog design
tags: dialog, datasource, granite-datasource, select, dynamic-options, servlet, sling, jcr-query, tags, pages, dam
---

## Dynamic Dialog Data with Granite DataSources

Granite DataSources provide a server-side mechanism for dynamically populating dialog field options (select dropdowns, radio groups, checkboxes, autocomplete lists) from JCR content, external APIs, or computed values.

### DataSource Concept and Architecture

A DataSource is a Sling resource type that produces a list of `Resource` objects at request time. Granite UI fields with a `<datasource>` child node invoke the DataSource's rendering script to populate their options.

**Flow:**
1. Author opens a component dialog
2. Granite UI renders the select/radio/checkbox field
3. The field finds a `<datasource>` child node
4. Sling resolves the datasource's `sling:resourceType` to a servlet or script
5. The servlet creates a `DataSource` object (list of synthetic resources)
6. The servlet sets `DataSource` as a request attribute
7. The field reads option text/value pairs from the DataSource resources

### Writing a DataSource Servlet

The standard pattern extends `SlingSafeMethodsServlet` and registers with a custom `sling:resourceType`:

```java
package com.myproject.core.datasources;

import com.adobe.granite.ui.components.ds.DataSource;
import com.adobe.granite.ui.components.ds.SimpleDataSource;
import com.adobe.granite.ui.components.ds.ValueMapResource;
import org.apache.sling.api.SlingHttpServletRequest;
import org.apache.sling.api.SlingHttpServletResponse;
import org.apache.sling.api.resource.Resource;
import org.apache.sling.api.resource.ResourceMetadata;
import org.apache.sling.api.resource.ResourceResolver;
import org.apache.sling.api.resource.ValueMap;
import org.apache.sling.api.servlets.SlingSafeMethodsServlet;
import org.apache.sling.api.wrappers.ValueMapDecorator;
import org.osgi.service.component.annotations.Component;

import javax.servlet.Servlet;
import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;

@Component(
    service = Servlet.class,
    property = {
        "sling.servlet.resourceTypes=myproject/datasources/countries",
        "sling.servlet.methods=GET",
        "sling.servlet.extensions=html"
    }
)
public class CountryDataSource extends SlingSafeMethodsServlet {

    @Override
    protected void doGet(SlingHttpServletRequest request, SlingHttpServletResponse response) {
        ResourceResolver resolver = request.getResourceResolver();
        List<Resource> options = new ArrayList<>();

        // Add a default empty option
        options.add(createOption(resolver, "", "-- Select Country --"));

        // Add country options
        options.add(createOption(resolver, "us", "United States"));
        options.add(createOption(resolver, "uk", "United Kingdom"));
        options.add(createOption(resolver, "de", "Germany"));
        options.add(createOption(resolver, "fr", "France"));
        options.add(createOption(resolver, "jp", "Japan"));

        // Set the DataSource on the request
        DataSource ds = new SimpleDataSource(options.iterator());
        request.setAttribute(DataSource.class.getName(), ds);
    }

    private Resource createOption(ResourceResolver resolver, String value, String text) {
        ValueMap vm = new ValueMapDecorator(new HashMap<>());
        vm.put("value", value);
        vm.put("text", text);
        return new ValueMapResource(resolver, new ResourceMetadata(), "nt:unstructured", vm);
    }
}
```

**Key classes:**

| Class | Package | Purpose |
|-------|---------|---------|
| `DataSource` | `com.adobe.granite.ui.components.ds` | Interface for datasource results |
| `SimpleDataSource` | `com.adobe.granite.ui.components.ds` | Wraps an `Iterator<Resource>` as a DataSource |
| `ValueMapResource` | `com.adobe.granite.ui.components.ds` | Synthetic resource with a ValueMap (text/value pairs) |

### Registering DataSource Resource Types

The servlet registration properties determine how Sling resolves the datasource:

```java
@Component(
    service = Servlet.class,
    property = {
        // This must match the sling:resourceType in the dialog's <datasource> node
        "sling.servlet.resourceTypes=myproject/datasources/pagelist",
        "sling.servlet.methods=GET",
        "sling.servlet.extensions=html"
    }
)
```

**Alternatively, use a JSP-based DataSource** at the resource type path:

```
/apps/myproject/datasources/pagelist/
├── .content.xml
└── pagelist.jsp
```

```xml
<!-- .content.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<jcr:root xmlns:sling="http://sling.apache.org/jcr/sling/1.0"
    xmlns:jcr="http://www.jcp.org/jcr/1.0"
    jcr:primaryType="sling:Folder"/>
```

```jsp
<%@include file="/libs/granite/ui/global.jsp" %>
<%@page session="false"
    import="java.util.*,
            org.apache.sling.api.resource.*,
            com.adobe.granite.ui.components.ds.*" %>
<%
    ResourceResolver resolver = resourceResolver;
    List<Resource> options = new ArrayList<>();

    Resource pages = resolver.getResource("/content/mysite/en");
    if (pages != null) {
        for (Resource page : pages.getChildren()) {
            ValueMap vm = new ValueMapDecorator(new HashMap<String, Object>());
            vm.put("value", page.getPath());
            vm.put("text", page.getValueMap().get("jcr:content/jcr:title", page.getName()));
            options.add(new ValueMapResource(resolver, new ResourceMetadata(), "nt:unstructured", vm));
        }
    }

    request.setAttribute(DataSource.class.getName(), new SimpleDataSource(options.iterator()));
%>
```

### Dialog Configuration for DataSource-Powered Fields

#### Select Dropdown

```xml
<country
    jcr:primaryType="nt:unstructured"
    sling:resourceType="granite/ui/components/coral/foundation/form/select"
    fieldLabel="Country"
    name="./country"
    emptyText="Select a country">
    <datasource
        jcr:primaryType="nt:unstructured"
        sling:resourceType="myproject/datasources/countries"/>
</country>
```

#### Radio Group

```xml
<layout
    jcr:primaryType="nt:unstructured"
    sling:resourceType="granite/ui/components/coral/foundation/form/radiogroup"
    fieldLabel="Layout"
    name="./layout">
    <datasource
        jcr:primaryType="nt:unstructured"
        sling:resourceType="myproject/datasources/layouts"/>
</radiogroup>
```

#### Checkbox Group

```xml
<features
    jcr:primaryType="nt:unstructured"
    sling:resourceType="granite/ui/components/coral/foundation/form/checkbox"
    fieldLabel="Features"
    name="./features">
    <datasource
        jcr:primaryType="nt:unstructured"
        sling:resourceType="myproject/datasources/features"/>
</features>
```

### DataSource with Parameters from Dialog Context

Pass configuration parameters to the DataSource via properties on the `<datasource>` node:

```xml
<category
    jcr:primaryType="nt:unstructured"
    sling:resourceType="granite/ui/components/coral/foundation/form/select"
    fieldLabel="Category"
    name="./category">
    <datasource
        jcr:primaryType="nt:unstructured"
        sling:resourceType="myproject/datasources/categories"
        rootPath="/content/mysite/categories"
        depth="{Long}2"
        includeEmpty="{Boolean}true"/>
</category>
```

**Reading parameters in the servlet:**

```java
@Component(
    service = Servlet.class,
    property = {
        "sling.servlet.resourceTypes=myproject/datasources/categories",
        "sling.servlet.methods=GET",
        "sling.servlet.extensions=html"
    }
)
public class CategoryDataSource extends SlingSafeMethodsServlet {

    @Override
    protected void doGet(SlingHttpServletRequest request, SlingHttpServletResponse response) {
        // The datasource node's properties are available via the request resource
        Resource datasourceResource = request.getResource();
        ValueMap dsProperties = datasourceResource.getValueMap();

        String rootPath = dsProperties.get("rootPath", "/content");
        int depth = dsProperties.get("depth", 1);
        boolean includeEmpty = dsProperties.get("includeEmpty", false);

        ResourceResolver resolver = request.getResourceResolver();
        List<Resource> options = new ArrayList<>();

        if (includeEmpty) {
            options.add(createOption(resolver, "", "-- None --"));
        }

        Resource root = resolver.getResource(rootPath);
        if (root != null) {
            collectCategories(resolver, root, options, 0, depth);
        }

        request.setAttribute(DataSource.class.getName(), new SimpleDataSource(options.iterator()));
    }

    private void collectCategories(ResourceResolver resolver, Resource parent,
                                    List<Resource> options, int currentDepth, int maxDepth) {
        if (currentDepth >= maxDepth) return;

        for (Resource child : parent.getChildren()) {
            if ("cq:Page".equals(child.getResourceType())) {
                String title = child.getValueMap().get("jcr:content/jcr:title", child.getName());
                String indent = currentDepth > 0 ? "\u00A0\u00A0".repeat(currentDepth) + "\u2514 " : "";
                options.add(createOption(resolver, child.getPath(), indent + title));
                collectCategories(resolver, child, options, currentDepth + 1, maxDepth);
            }
        }
    }

    private Resource createOption(ResourceResolver resolver, String value, String text) {
        ValueMap vm = new ValueMapDecorator(new HashMap<>());
        vm.put("value", value);
        vm.put("text", text);
        return new ValueMapResource(resolver, new ResourceMetadata(), "nt:unstructured", vm);
    }
}
```

### Accessing the Component Context

To access the component being edited (current page, component path, etc.), use the request suffix and `ExpressionHelper`:

```java
@Component(
    service = Servlet.class,
    property = {
        "sling.servlet.resourceTypes=myproject/datasources/contextaware",
        "sling.servlet.methods=GET",
        "sling.servlet.extensions=html"
    }
)
public class ContextAwareDataSource extends SlingSafeMethodsServlet {

    @Override
    protected void doGet(SlingHttpServletRequest request, SlingHttpServletResponse response) {
        ResourceResolver resolver = request.getResourceResolver();

        // The content resource path is available via the request suffix
        String contentPath = request.getRequestPathInfo().getSuffix();
        Resource contentResource = null;
        if (contentPath != null) {
            contentResource = resolver.getResource(contentPath);
        }

        // Use ExpressionHelper for EL expressions in datasource properties
        ExpressionHelper expressionHelper = new ExpressionHelper(
            cmp.getExpressionResolver(), request);
        Resource dsResource = request.getResource();
        String rootPath = expressionHelper.getString(
            dsResource.getValueMap().get("rootPath", String.class));

        List<Resource> options = new ArrayList<>();

        // Build options based on the current component's context
        if (contentResource != null) {
            // Get the current page
            PageManager pageManager = resolver.adaptTo(PageManager.class);
            Page currentPage = pageManager.getContainingPage(contentResource);

            if (currentPage != null) {
                // Populate options based on sibling pages, parent content, etc.
                Page parent = currentPage.getParent();
                if (parent != null) {
                    for (Iterator<Page> it = parent.listChildren(); it.hasNext();) {
                        Page sibling = it.next();
                        options.add(createOption(resolver, sibling.getPath(), sibling.getTitle()));
                    }
                }
            }
        }

        request.setAttribute(DataSource.class.getName(), new SimpleDataSource(options.iterator()));
    }

    private Resource createOption(ResourceResolver resolver, String value, String text) {
        ValueMap vm = new ValueMapDecorator(new HashMap<>());
        vm.put("value", value);
        vm.put("text", text);
        return new ValueMapResource(resolver, new ResourceMetadata(), "nt:unstructured", vm);
    }
}
```

### Caching DataSource Results

For expensive DataSource computations, implement caching:

```java
@Component(
    service = Servlet.class,
    property = {
        "sling.servlet.resourceTypes=myproject/datasources/externalapi",
        "sling.servlet.methods=GET",
        "sling.servlet.extensions=html"
    }
)
public class ExternalApiDataSource extends SlingSafeMethodsServlet {

    private static final long CACHE_TTL_MS = 300_000; // 5 minutes

    // Simple in-memory cache — for production, use a proper cache framework
    private volatile List<Map<String, String>> cachedOptions;
    private volatile long cacheTimestamp;

    @Reference
    private ExternalApiService apiService;

    @Override
    protected void doGet(SlingHttpServletRequest request, SlingHttpServletResponse response) {
        ResourceResolver resolver = request.getResourceResolver();
        List<Resource> options = new ArrayList<>();

        List<Map<String, String>> apiOptions = getCachedOptions();
        for (Map<String, String> opt : apiOptions) {
            options.add(createOption(resolver, opt.get("id"), opt.get("name")));
        }

        request.setAttribute(DataSource.class.getName(), new SimpleDataSource(options.iterator()));
    }

    private synchronized List<Map<String, String>> getCachedOptions() {
        long now = System.currentTimeMillis();
        if (cachedOptions == null || (now - cacheTimestamp) > CACHE_TTL_MS) {
            try {
                cachedOptions = apiService.fetchOptions();
                cacheTimestamp = now;
            } catch (Exception e) {
                if (cachedOptions == null) {
                    cachedOptions = Collections.emptyList();
                }
                // Return stale cache on failure
            }
        }
        return cachedOptions;
    }

    private Resource createOption(ResourceResolver resolver, String value, String text) {
        ValueMap vm = new ValueMapDecorator(new HashMap<>());
        vm.put("value", value);
        vm.put("text", text);
        return new ValueMapResource(resolver, new ResourceMetadata(), "nt:unstructured", vm);
    }
}
```

### Common DataSource Examples

#### Tag List DataSource

Populate a select field with tags from a specific namespace:

```java
@Component(
    service = Servlet.class,
    property = {
        "sling.servlet.resourceTypes=myproject/datasources/tags",
        "sling.servlet.methods=GET",
        "sling.servlet.extensions=html"
    }
)
public class TagDataSource extends SlingSafeMethodsServlet {

    @Reference
    private TagManager tagManager;

    @Override
    protected void doGet(SlingHttpServletRequest request, SlingHttpServletResponse response) {
        ResourceResolver resolver = request.getResourceResolver();
        Resource dsResource = request.getResource();
        String namespace = dsResource.getValueMap().get("namespace", "mysite");

        List<Resource> options = new ArrayList<>();
        options.add(createOption(resolver, "", "-- Select Tag --"));

        TagManager tm = resolver.adaptTo(TagManager.class);
        if (tm != null) {
            Tag namespaceTag = tm.resolve(namespace + ":");
            if (namespaceTag != null) {
                for (Iterator<Tag> tags = namespaceTag.listChildren(); tags.hasNext();) {
                    Tag tag = tags.next();
                    options.add(createOption(resolver, tag.getTagID(), tag.getTitle()));
                }
            }
        }

        request.setAttribute(DataSource.class.getName(), new SimpleDataSource(options.iterator()));
    }

    private Resource createOption(ResourceResolver resolver, String value, String text) {
        ValueMap vm = new ValueMapDecorator(new HashMap<>());
        vm.put("value", value);
        vm.put("text", text);
        return new ValueMapResource(resolver, new ResourceMetadata(), "nt:unstructured", vm);
    }
}
```

**Dialog usage:**

```xml
<tag
    jcr:primaryType="nt:unstructured"
    sling:resourceType="granite/ui/components/coral/foundation/form/select"
    fieldLabel="Category Tag"
    name="./categoryTag">
    <datasource
        jcr:primaryType="nt:unstructured"
        sling:resourceType="myproject/datasources/tags"
        namespace="mysite/categories"/>
</tag>
```

#### Page List DataSource

List child pages of a configurable root as select options:

```java
@Component(
    service = Servlet.class,
    property = {
        "sling.servlet.resourceTypes=myproject/datasources/pagelist",
        "sling.servlet.methods=GET",
        "sling.servlet.extensions=html"
    }
)
public class PageListDataSource extends SlingSafeMethodsServlet {

    @Override
    protected void doGet(SlingHttpServletRequest request, SlingHttpServletResponse response) {
        ResourceResolver resolver = request.getResourceResolver();
        Resource dsResource = request.getResource();

        String rootPath = dsResource.getValueMap().get("rootPath", "/content");
        String resourceType = dsResource.getValueMap().get("filterResourceType", String.class);

        PageManager pageManager = resolver.adaptTo(PageManager.class);
        Page rootPage = pageManager.getPage(rootPath);

        List<Resource> options = new ArrayList<>();
        options.add(createOption(resolver, "", "-- Select Page --"));

        if (rootPage != null) {
            for (Iterator<Page> it = rootPage.listChildren(
                    new PageFilter(false, false), true); it.hasNext();) {
                Page page = it.next();

                // Optional: filter by resource type
                if (resourceType != null && !resourceType.equals(
                        page.getProperties().get("sling:resourceType", ""))) {
                    continue;
                }

                // Indent based on depth
                int depth = StringUtils.countMatches(
                    page.getPath().replace(rootPath, ""), "/") - 1;
                String indent = depth > 0 ? "\u00A0\u00A0".repeat(depth) : "";

                options.add(createOption(resolver, page.getPath(),
                    indent + page.getTitle()));
            }
        }

        request.setAttribute(DataSource.class.getName(), new SimpleDataSource(options.iterator()));
    }

    private Resource createOption(ResourceResolver resolver, String value, String text) {
        ValueMap vm = new ValueMapDecorator(new HashMap<>());
        vm.put("value", value);
        vm.put("text", text);
        return new ValueMapResource(resolver, new ResourceMetadata(), "nt:unstructured", vm);
    }
}
```

#### DAM Folder Contents DataSource

List assets from a DAM folder:

```java
@Component(
    service = Servlet.class,
    property = {
        "sling.servlet.resourceTypes=myproject/datasources/damassets",
        "sling.servlet.methods=GET",
        "sling.servlet.extensions=html"
    }
)
public class DamAssetDataSource extends SlingSafeMethodsServlet {

    @Override
    protected void doGet(SlingHttpServletRequest request, SlingHttpServletResponse response) {
        ResourceResolver resolver = request.getResourceResolver();
        Resource dsResource = request.getResource();

        String folderPath = dsResource.getValueMap().get("folderPath", "/content/dam");
        String mimeTypeFilter = dsResource.getValueMap().get("mimeType", String.class);

        List<Resource> options = new ArrayList<>();

        Resource folder = resolver.getResource(folderPath);
        if (folder != null) {
            for (Resource child : folder.getChildren()) {
                Asset asset = child.adaptTo(Asset.class);
                if (asset != null) {
                    // Optional MIME type filter
                    if (mimeTypeFilter != null &&
                        !asset.getMimeType().startsWith(mimeTypeFilter)) {
                        continue;
                    }

                    String title = asset.getMetadataValue("dc:title");
                    if (title == null || title.isEmpty()) {
                        title = asset.getName();
                    }

                    options.add(createOption(resolver, asset.getPath(), title));
                }
            }
        }

        request.setAttribute(DataSource.class.getName(), new SimpleDataSource(options.iterator()));
    }

    private Resource createOption(ResourceResolver resolver, String value, String text) {
        ValueMap vm = new ValueMapDecorator(new HashMap<>());
        vm.put("value", value);
        vm.put("text", text);
        return new ValueMapResource(resolver, new ResourceMetadata(), "nt:unstructured", vm);
    }
}
```

**Dialog usage:**

```xml
<document
    jcr:primaryType="nt:unstructured"
    sling:resourceType="granite/ui/components/coral/foundation/form/select"
    fieldLabel="PDF Document"
    name="./documentPath">
    <datasource
        jcr:primaryType="nt:unstructured"
        sling:resourceType="myproject/datasources/damassets"
        folderPath="/content/dam/mysite/documents"
        mimeType="application/pdf"/>
</document>
```

#### i18n Keys DataSource

Populate options from i18n dictionaries:

```java
@Component(
    service = Servlet.class,
    property = {
        "sling.servlet.resourceTypes=myproject/datasources/i18nkeys",
        "sling.servlet.methods=GET",
        "sling.servlet.extensions=html"
    }
)
public class I18nKeysDataSource extends SlingSafeMethodsServlet {

    @Override
    protected void doGet(SlingHttpServletRequest request, SlingHttpServletResponse response) {
        ResourceResolver resolver = request.getResourceResolver();
        Resource dsResource = request.getResource();
        String prefix = dsResource.getValueMap().get("keyPrefix", "myproject.labels.");

        Locale locale = request.getLocale();
        ResourceBundle bundle = request.getResourceBundle(locale);

        List<Resource> options = new ArrayList<>();

        if (bundle != null) {
            Enumeration<String> keys = bundle.getKeys();
            while (keys.hasMoreElements()) {
                String key = keys.nextElement();
                if (key.startsWith(prefix)) {
                    String label = bundle.getString(key);
                    options.add(createOption(resolver, key, label));
                }
            }
        }

        // Sort by display text
        options.sort((a, b) -> {
            String textA = a.getValueMap().get("text", "");
            String textB = b.getValueMap().get("text", "");
            return textA.compareToIgnoreCase(textB);
        });

        request.setAttribute(DataSource.class.getName(), new SimpleDataSource(options.iterator()));
    }

    private Resource createOption(ResourceResolver resolver, String value, String text) {
        ValueMap vm = new ValueMapDecorator(new HashMap<>());
        vm.put("value", value);
        vm.put("text", text);
        return new ValueMapResource(resolver, new ResourceMetadata(), "nt:unstructured", vm);
    }
}
```

### DataSource for Pathbrowser Root Filtering

Create a DataSource that filters pathfield suggestions based on a configurable root:

```xml
<targetPage
    jcr:primaryType="nt:unstructured"
    sling:resourceType="granite/ui/components/coral/foundation/form/pathfield"
    fieldLabel="Target Page"
    name="./targetPage"
    rootPath="/content/mysite"
    filter="hierarchyNotFile"
    pickerSrc="/apps/myproject/datasources/filteredpages{.offset,limit}.html"
    suggestionSrc="/apps/myproject/datasources/filteredpages/suggestions{.offset,limit}.html{+query}"/>
```

### Built-in AEM DataSources

AEM provides several out-of-the-box DataSource resource types:

| Resource Type | Purpose |
|---------------|---------|
| `cq/gui/components/common/datasources/tags` | AEM tags |
| `dam/cfm/components/datasources/elements` | Content Fragment elements |
| `dam/cfm/components/datasources/variations` | Content Fragment variations |
| `cq/gui/components/common/datasources/allowedcomponents` | Allowed components for a parsys |
| `granite/ui/components/coral/foundation/form/responses/datasources` | Form response types |

### Anti-Patterns

#### N+1 Queries

```java
// BAD: Individual query per option
for (String id : ids) {
    Resource r = resolver.getResource("/content/items/" + id);
    // This causes N+1 repository reads
}

// GOOD: Single query for all items
String query = "SELECT * FROM [nt:unstructured] WHERE ISDESCENDANTNODE('/content/items')";
Iterator<Resource> results = resolver.findResources(query, Query.JCR_SQL2);
```

#### Uncached External Calls

```java
// BAD: HTTP call on every dialog open
@Override
protected void doGet(...) {
    // This runs every time ANY author opens this component dialog
    HttpResponse response = httpClient.execute(new HttpGet("https://api.example.com/options"));
    // Parse and create options...
}

// GOOD: Cache with TTL (see Caching DataSource Results section above)
```

#### Unbounded Result Sets

```java
// BAD: Loading all assets in DAM
Resource dam = resolver.getResource("/content/dam");
for (Resource asset : dam.getChildren()) { ... } // Could be thousands

// GOOD: Limit results and paginate
String query = "SELECT * FROM [dam:Asset] WHERE ISDESCENDANTNODE('/content/dam/mysite') "
    + "ORDER BY [jcr:content/jcr:lastModified] DESC";
// Use QueryBuilder with limit
Map<String, String> params = new HashMap<>();
params.put("path", "/content/dam/mysite");
params.put("type", "dam:Asset");
params.put("p.limit", "50");
Query qb = queryBuilder.createQuery(PredicateGroup.create(params), session);
```

#### Missing Error Handling

```java
// BAD: No fallback if resource is null
Resource root = resolver.getResource(rootPath);
for (Resource child : root.getChildren()) { ... } // NPE if root is null

// GOOD: Null-safe with empty DataSource fallback
Resource root = resolver.getResource(rootPath);
if (root == null) {
    request.setAttribute(DataSource.class.getName(),
        new SimpleDataSource(Collections.emptyIterator()));
    return;
}
```

### Testing DataSources

Write unit tests using Sling Mocks:

```java
@ExtendWith(SlingContextExtension.class)
class CountryDataSourceTest {

    private final SlingContext context = new SlingContext(ResourceResolverType.JCR_MOCK);

    @Test
    void testDataSourceReturnsCountries() throws Exception {
        // Set up the datasource resource with properties
        context.create().resource("/apps/ds", "rootPath", "/content/countries");

        CountryDataSource servlet = new CountryDataSource();
        context.request().setResource(context.resourceResolver().getResource("/apps/ds"));

        servlet.doGet(context.request(), context.response());

        DataSource ds = (DataSource) context.request().getAttribute(DataSource.class.getName());
        assertNotNull(ds);

        List<Resource> options = new ArrayList<>();
        ds.iterator().forEachRemaining(options::add);

        assertFalse(options.isEmpty());
        assertEquals("us", options.get(1).getValueMap().get("value"));
    }
}
```
