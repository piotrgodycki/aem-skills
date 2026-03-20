---
title: Sling Models for Frontend Developers
impact: HIGH
impactDescription: Sling Models bridge backend data to HTL templates — incorrect annotations or patterns cause silent null values, broken components, and JSON export failures
tags: sling-models, htl, annotations, json-exporter, core-components, delegation, testing, valueMapValue
---

## Sling Models for Frontend Developers

Sling Models are annotation-driven Java POJOs that map AEM resource data (JCR properties) to Java objects. They are the recommended Use-API for backing HTL components, replacing the legacy WCMUsePojo approach.

### Basic Sling Model Structure

#### @Model Annotation

```java
@Model(
    adaptables = SlingHttpServletRequest.class,   // What this model adapts from
    adapters = MyComponent.class,                  // Interface exposed to HTL
    resourceType = "myproject/components/mycomp",  // Ties model to component resourceType
    defaultInjectionStrategy = DefaultInjectionStrategy.OPTIONAL // Fields default to optional
)
public class MyComponentImpl implements MyComponent {
    // ...
}
```

**Key parameters**:

| Parameter | Purpose |
|-----------|---------|
| `adaptables` | Source object type: `SlingHttpServletRequest.class` (preferred, gives access to request) or `Resource.class` (simpler, no request access) |
| `adapters` | The interface this model registers under — HTL uses this interface, not the impl class |
| `resourceType` | Associates the model with a specific component sling:resourceType for automatic resolution |
| `defaultInjectionStrategy` | `OPTIONAL` means null fields do not cause errors; `REQUIRED` (default) throws exceptions on missing values |

#### Adaptables: Request vs Resource

```java
// PREFERRED: Request-adaptable — access to request, selectors, session, etc.
@Model(adaptables = SlingHttpServletRequest.class, adapters = Hero.class,
       resourceType = "myproject/components/hero",
       defaultInjectionStrategy = DefaultInjectionStrategy.OPTIONAL)
public class HeroImpl implements Hero { }

// SIMPLER: Resource-adaptable — sufficient when you only need JCR properties
@Model(adaptables = Resource.class, adapters = Hero.class,
       resourceType = "myproject/components/hero",
       defaultInjectionStrategy = DefaultInjectionStrategy.OPTIONAL)
public class HeroImpl implements Hero { }
```

Use `SlingHttpServletRequest.class` when you need access to request attributes, selectors, WCM mode, other Sling Models via ModelFactory, or OSGi services.

### Injecting Dialog Values

#### @ValueMapValue — JCR Properties

```java
@Model(adaptables = Resource.class,
       defaultInjectionStrategy = DefaultInjectionStrategy.OPTIONAL)
public class TextComponentImpl implements TextComponent {

    // Injects jcr:title from the component resource's ValueMap
    @ValueMapValue(name = "jcr:title")
    private String title;

    // Field name matches property name — no name param needed
    @ValueMapValue
    private String description;

    // Default value when property is null
    @ValueMapValue
    @Default(values = "Read More")
    private String ctaLabel;

    // Boolean properties
    @ValueMapValue
    @Default(booleanValues = false)
    private boolean hideInNav;

    // Typed injection
    @ValueMapValue
    private Calendar startDate;

    // Path reference stored as String
    @ValueMapValue
    private String imagePath;
}
```

#### @ChildResource — Nested Resources (Multifields)

```java
// Single child resource
@ChildResource
private Resource image;

// List of child resources (from multifield)
@ChildResource
private List<Resource> items;
```

#### @Self — The Adaptable Itself

```java
// Inject the request or resource itself
@Self
private SlingHttpServletRequest request;

@Self
private Resource resource;

// Adapt self to another Sling Model
@Self
private Image image;  // Adapts current request to Image model
```

#### @OSGiService — Inject OSGi Services

```java
// OSGi service injection (cannot use @Reference in Sling Models)
@OSGiService
private ModelFactory modelFactory;

@OSGiService
private PageManagerFactory pageManagerFactory;

@OSGiService
private ResourceResolverFactory resolverFactory;

@OSGiService
private Externalizer externalizer;
```

#### @SlingObject — Sling-Specific Objects

```java
@SlingObject
private Resource resource;

@SlingObject
private ResourceResolver resourceResolver;

@SlingObject
private SlingHttpServletRequest request;

@SlingObject
private SlingHttpServletResponse response;
```

#### @ScriptVariable — HTL Global Objects

```java
// Inject any HTL global binding object
@ScriptVariable
private Page currentPage;

@ScriptVariable
private Designer designer;

@ScriptVariable
private Style currentStyle;

@ScriptVariable
private ComponentContext componentContext;
```

#### @RequestAttribute — Parameters from HTL

```java
// Receive parameters passed from HTL data-sly-use
@RequestAttribute(name = "title")
private String title;

@RequestAttribute(name = "showDetails")
@Default(booleanValues = false)
private boolean showDetails;
```

HTL side:

```html
<sly data-sly-use.card="${'com.myproject.core.models.Card' @
    title='My Card Title',
    showDetails=true}"/>
```

### Exposing Data for HTL Templates

HTL accesses Sling Model data through public getter methods. HTL shortens getter names: `getTitle()` becomes `${model.title}`, `isEmpty()` becomes `${model.empty}`.

#### Interface-Based Pattern (Recommended)

```java
// Interface — this is what HTL sees
public interface HeroComponent {
    String getTitle();
    String getDescription();
    String getImagePath();
    String getLinkURL();
    boolean isEmpty();
}

// Implementation
@Model(adaptables = SlingHttpServletRequest.class,
       adapters = HeroComponent.class,
       resourceType = "myproject/components/hero",
       defaultInjectionStrategy = DefaultInjectionStrategy.OPTIONAL)
public class HeroComponentImpl implements HeroComponent {

    @ValueMapValue
    private String title;

    @ValueMapValue
    private String description;

    @ValueMapValue(name = "fileReference")
    private String imagePath;

    @ValueMapValue
    private String linkURL;

    @Override
    public String getTitle() { return title; }

    @Override
    public String getDescription() { return description; }

    @Override
    public String getImagePath() { return imagePath; }

    @Override
    public String getLinkURL() { return linkURL; }

    @Override
    public boolean isEmpty() {
        return StringUtils.isBlank(title) && StringUtils.isBlank(imagePath);
    }
}
```

HTL usage:

```html
<sly data-sly-use.hero="com.myproject.core.models.HeroComponent"/>
<div data-sly-test.hasContent="${!hero.empty}" class="cmp-hero">
    <h1 class="cmp-hero__title">${hero.title}</h1>
    <p class="cmp-hero__description" data-sly-test="${hero.description}">
        ${hero.description}
    </p>
    <img data-sly-test="${hero.imagePath}"
         class="cmp-hero__image" src="${hero.imagePath}" alt="${hero.title}"/>
    <a data-sly-test="${hero.linkURL}"
       class="cmp-hero__cta" href="${hero.linkURL}">Learn More</a>
</div>
<sly data-sly-test="${!hasContent && wcmmode.edit}">
    <div class="cq-placeholder" data-emptytext="${component.title}"></div>
</sly>
```

### @PostConstruct Initialization

Use `@PostConstruct` for logic that depends on multiple injected fields. It runs once after all injection is complete.

```java
@Model(adaptables = SlingHttpServletRequest.class,
       adapters = Byline.class,
       resourceType = "myproject/components/byline",
       defaultInjectionStrategy = DefaultInjectionStrategy.OPTIONAL)
public class BylineImpl implements Byline {

    @ValueMapValue
    private String name;

    @ValueMapValue
    private List<String> occupations;

    @Self
    private SlingHttpServletRequest request;

    @OSGiService
    private ModelFactory modelFactory;

    private Image image;

    @PostConstruct
    private void init() {
        // Use ModelFactory to get secondary models from the same request
        image = modelFactory.getModelFromWrappedRequest(
            request, request.getResource(), Image.class);
    }

    @Override
    public String getName() { return name; }

    @Override
    public List<String> getOccupations() {
        return occupations != null ? Collections.unmodifiableList(occupations)
                                   : Collections.emptyList();
    }

    @Override
    public boolean isEmpty() {
        if (StringUtils.isBlank(name)) return true;
        if (image == null || StringUtils.isBlank(image.getSrc())) return true;
        if (occupations == null || occupations.isEmpty()) return true;
        return false;
    }
}
```

### Multifield Value Injection Patterns

#### Simple Multifield (List of Strings)

```java
// Tags or simple values stored as String[]
@ValueMapValue
private List<String> tags;

// Or as String array
@ValueMapValue
private String[] categories;
```

#### Composite Multifield (List of Child Resources)

Each multifield entry is stored as a child resource node.

```java
@Model(adaptables = SlingHttpServletRequest.class,
       adapters = CardList.class,
       resourceType = "myproject/components/cardlist",
       defaultInjectionStrategy = DefaultInjectionStrategy.OPTIONAL)
public class CardListImpl implements CardList {

    @ChildResource
    private List<Resource> cards;

    @Override
    public List<CardItem> getCardItems() {
        if (cards == null) return Collections.emptyList();
        return cards.stream()
            .map(r -> r.adaptTo(CardItem.class))
            .filter(Objects::nonNull)
            .collect(Collectors.toList());
    }
}

// Child item model (adaptable from Resource)
@Model(adaptables = Resource.class,
       defaultInjectionStrategy = DefaultInjectionStrategy.OPTIONAL)
public class CardItem {

    @ValueMapValue
    private String title;

    @ValueMapValue
    private String description;

    @ValueMapValue
    private String linkURL;

    public String getTitle() { return title; }
    public String getDescription() { return description; }
    public String getLinkURL() { return linkURL; }
}
```

### Exporting as JSON for Headless (@Exporter)

Sling Model Exporter allows components to expose JSON representations for headless consumers, SPAs, or JavaScript-driven frontend components.

```java
@Model(adaptables = SlingHttpServletRequest.class,
       adapters = { EventComponent.class, ComponentExporter.class },
       resourceType = "myproject/components/event",
       defaultInjectionStrategy = DefaultInjectionStrategy.OPTIONAL)
@Exporter(name = ExporterConstants.SLING_MODEL_EXPORTER_NAME,
          extensions = ExporterConstants.SLING_MODEL_EXTENSION)
public class EventComponentImpl implements EventComponent, ComponentExporter {

    @ValueMapValue
    private String title;

    @ValueMapValue
    private String location;

    @ValueMapValue
    private Calendar eventDate;

    @Override
    public String getTitle() { return title; }

    @Override
    public String getLocation() { return location; }

    @Override
    public Calendar getEventDate() { return eventDate; }

    // Required by ComponentExporter
    @Override
    public String getExportedType() {
        return "myproject/components/event";
    }
}
```

Access JSON at: `/content/mysite/page/jcr:content/event.model.json`

Output:

```json
{
    "title": "Annual Conference",
    "location": "New York",
    "eventDate": "2025-06-15T09:00:00.000Z",
    ":type": "myproject/components/event"
}
```

#### Controlling JSON Output with Jackson Annotations

```java
@Model(adaptables = SlingHttpServletRequest.class,
       adapters = { MyComponent.class, ComponentExporter.class },
       resourceType = "myproject/components/mycomp")
@Exporter(name = ExporterConstants.SLING_MODEL_EXPORTER_NAME,
          extensions = ExporterConstants.SLING_MODEL_EXTENSION,
          options = {
              @ExporterOption(name = "MapperFeature.SORT_PROPERTIES_ALPHABETICALLY",
                              value = "true"),
              @ExporterOption(name = "SerializationFeature.WRITE_DATES_AS_TIMESTAMPS",
                              value = "false")
          })
public class MyComponentImpl implements MyComponent, ComponentExporter {

    @JsonIgnore        // Exclude from JSON output
    private String internalId;

    @JsonProperty("headline")   // Rename in JSON
    public String getTitle() { return title; }
}
```

### Sling Model Delegation for Core Component Extension

Use the delegation pattern with `@Via(type = ResourceSuperType.class)` to extend Core Components without modifying their code.

```java
@Model(adaptables = SlingHttpServletRequest.class,
       adapters = Title.class,
       resourceType = "myproject/components/title")
public class CustomTitleImpl implements Title {

    // Delegate to the Core Component's model via sling:resourceSuperType
    @Self
    @Via(type = ResourceSuperType.class)
    private Title title;

    @ScriptVariable
    private Page currentPage;

    // Override: always use the page title
    @Override
    public String getText() {
        return currentPage.getTitle();
    }

    // Delegate everything else to the Core Component
    @Override
    public String getType() {
        return title.getType();
    }

    @Override
    public String getLinkURL() {
        return title.getLinkURL();
    }

    @Override
    public boolean isLinkDisabled() {
        return title.isLinkDisabled();
    }

    @Override
    public String getId() {
        return title.getId();
    }

    @Override
    public ComponentData getData() {
        return title.getData();
    }

    @Override
    public String getExportedType() {
        return "myproject/components/title";
    }

    @Override
    public String getAppliedCssClasses() {
        return title.getAppliedCssClasses();
    }
}
```

The proxy component at `/apps/myproject/components/title` must have:

```
sling:resourceSuperType = "core/wcm/components/title/v3/title"
```

### Injecting Page Properties, Request Attributes, and Selectors

```java
@Model(adaptables = SlingHttpServletRequest.class,
       defaultInjectionStrategy = DefaultInjectionStrategy.OPTIONAL)
public class PageMetaImpl implements PageMeta {

    // Inject page properties (not component properties)
    @ScriptVariable
    private Page currentPage;

    // Access request selectors
    @Self
    private SlingHttpServletRequest request;

    public String getPageTitle() {
        return currentPage.getPageTitle() != null
            ? currentPage.getPageTitle()
            : currentPage.getTitle();
    }

    public String getPageDescription() {
        ValueMap pageProps = currentPage.getProperties();
        return pageProps.get("jcr:description", String.class);
    }

    public String[] getSelectors() {
        return request.getRequestPathInfo().getSelectors();
    }

    public boolean isMobileSelector() {
        String[] selectors = request.getRequestPathInfo().getSelectors();
        return Arrays.asList(selectors).contains("mobile");
    }

    public String getTemplatePath() {
        return currentPage.getProperties().get("cq:template", String.class);
    }

    public List<Tag> getPageTags() {
        Tag[] tags = currentPage.getTags();
        return tags != null ? Arrays.asList(tags) : Collections.emptyList();
    }
}
```

### Common Patterns

#### Getting Child Pages (Navigation)

```java
@Model(adaptables = SlingHttpServletRequest.class,
       adapters = Navigation.class,
       resourceType = "myproject/components/navigation",
       defaultInjectionStrategy = DefaultInjectionStrategy.OPTIONAL)
public class NavigationImpl implements Navigation {

    @ScriptVariable
    private Page currentPage;

    @Override
    public List<NavItem> getItems() {
        Page rootPage = currentPage.getAbsoluteParent(2); // site root
        if (rootPage == null) return Collections.emptyList();

        List<NavItem> items = new ArrayList<>();
        Iterator<Page> children = rootPage.listChildren();
        while (children.hasNext()) {
            Page child = children.next();
            if (!child.isHideInNav()) {
                items.add(new NavItem(
                    child.getTitle(),
                    child.getPath() + ".html",
                    child.getPath().equals(currentPage.getPath())
                ));
            }
        }
        return items;
    }
}
```

#### Resolving DAM Asset References

```java
@Model(adaptables = SlingHttpServletRequest.class,
       defaultInjectionStrategy = DefaultInjectionStrategy.OPTIONAL)
public class AssetInfoImpl implements AssetInfo {

    @ValueMapValue(name = "fileReference")
    private String assetPath;

    @SlingObject
    private ResourceResolver resourceResolver;

    public String getAssetTitle() {
        if (StringUtils.isBlank(assetPath)) return null;
        Resource assetResource = resourceResolver.getResource(assetPath);
        if (assetResource == null) return null;
        Asset asset = assetResource.adaptTo(Asset.class);
        return asset != null ? asset.getMetadataValue("dc:title") : null;
    }
}
```

### Testing Sling Models

Use `io.wcm.testing.mock.aem.junit5.AemContext` for unit testing Sling Models.

```java
@ExtendWith(AemContextExtension.class)
class HeroComponentImplTest {

    private final AemContext ctx = new AemContext();

    @BeforeEach
    void setUp() {
        // Register the Sling Model
        ctx.addModelsForClasses(HeroComponentImpl.class);

        // Load test content
        ctx.load().json("/hero-content.json", "/content/mysite/page/jcr:content/hero");
    }

    @Test
    void testGetTitle() {
        // Set the current resource
        ctx.currentResource("/content/mysite/page/jcr:content/hero");

        // Adapt to the model
        HeroComponent model = ctx.request().adaptTo(HeroComponent.class);

        assertNotNull(model);
        assertEquals("Expected Title", model.getTitle());
    }

    @Test
    void testIsEmptyWhenNoTitle() {
        // Create resource without title
        ctx.create().resource("/content/mysite/page/jcr:content/empty-hero",
            "sling:resourceType", "myproject/components/hero");
        ctx.currentResource("/content/mysite/page/jcr:content/empty-hero");

        HeroComponent model = ctx.request().adaptTo(HeroComponent.class);

        assertNotNull(model);
        assertTrue(model.isEmpty());
    }

    @Test
    void testGetOccupationsJoined() {
        ctx.currentResource("/content/mysite/page/jcr:content/hero");
        ctx.request().setAttribute("occupations",
            new String[]{"Developer", "Designer"});

        HeroComponent model = ctx.request().adaptTo(HeroComponent.class);
        assertNotNull(model.getOccupations());
        assertFalse(model.getOccupations().isEmpty());
    }
}
```

Test content JSON (`/hero-content.json`):

```json
{
    "jcr:primaryType": "nt:unstructured",
    "sling:resourceType": "myproject/components/hero",
    "jcr:title": "Expected Title",
    "description": "Hero description text",
    "fileReference": "/content/dam/mysite/hero.jpg"
}
```

### Anti-Patterns to Avoid

#### Do Not Put Business Logic in HTL

```html
<!-- WRONG: Complex logic in HTL -->
<sly data-sly-list.page="${currentPage.listChildren}">
    <sly data-sly-test="${page.properties.hideInNav != 'true'
        && page.properties.jcr:title
        && page.depth < 5}">
        <li>${page.title}</li>
    </sly>
</sly>

<!-- CORRECT: Delegate to Sling Model -->
<sly data-sly-use.nav="com.myproject.core.models.Navigation"/>
<li data-sly-repeat.page="${nav.visiblePages}">${page.title}</li>
```

#### Do Not Use WCMUsePojo in New Code

```java
// WRONG: Legacy pattern — no dependency injection, no testability
public class MyHelper extends WCMUsePojo {
    @Override
    public void activate() throws Exception {
        // manual property lookup
    }
}

// CORRECT: Sling Model — injectable, testable, cacheable
@Model(adaptables = SlingHttpServletRequest.class)
public class MyComponentImpl implements MyComponent {
    @ValueMapValue
    private String title;
}
```

#### Do Not Inject Too Many Services

```java
// WRONG: Model doing too much — split responsibilities
@Model(adaptables = SlingHttpServletRequest.class)
public class KitchenSinkModel {
    @OSGiService private QueryBuilder queryBuilder;
    @OSGiService private Replicator replicator;
    @OSGiService private WorkflowService workflowService;
    @OSGiService private Externalizer externalizer;
    @OSGiService private TagManager tagManager;
    @OSGiService private ModelFactory modelFactory;
    @OSGiService private SlingSettingsService settingsService;
    // ... too many services = too many responsibilities
}

// CORRECT: Single responsibility — one model per concern
@Model(adaptables = SlingHttpServletRequest.class)
public class ArticleComponentImpl implements ArticleComponent {
    @ValueMapValue private String title;
    @ValueMapValue private String author;
    @ScriptVariable private Page currentPage;
    // Focused on article data only
}
```

#### Do Not Adapt from Resource When You Need the Request

```java
// WRONG: Cannot inject request-scoped objects or use ModelFactory
@Model(adaptables = Resource.class)
public class MyModel {
    // @OSGiService ModelFactory modelFactory;  <-- will fail at runtime
    // Cannot access selectors, WCM mode, session, etc.
}

// CORRECT: Use Request adaptable when you need request scope
@Model(adaptables = SlingHttpServletRequest.class)
public class MyModel {
    @OSGiService
    private ModelFactory modelFactory;  // works
}
```

#### Do Not Forget defaultInjectionStrategy

```java
// WRONG: Default is REQUIRED — any missing property throws an exception
@Model(adaptables = Resource.class)
public class FragileModel {
    @ValueMapValue
    private String optionalField;  // Exception if missing!
}

// CORRECT: Use OPTIONAL when fields may be absent
@Model(adaptables = Resource.class,
       defaultInjectionStrategy = DefaultInjectionStrategy.OPTIONAL)
public class SafeModel {
    @ValueMapValue
    private String optionalField;  // null if missing, no error
}
```
