---
title: HTL (HTML Template Language) Best Practices
impact: CRITICAL
impactDescription: HTL is the mandatory server-side template language for AEM — incorrect usage causes XSS vulnerabilities, rendering failures, and performance degradation
tags: htl, sightly, xss, templating, data-sly, use-api, sling-models, expressions
---

## HTL (HTML Template Language) Best Practices

HTL (formerly Sightly) is the preferred and recommended server-side template system for HTML in Adobe Experience Manager. It compiles into Java Servlets, with expressions and block statements evaluated entirely server-side.

### Expression Language

HTL expressions use `${}` delimiters for runtime-evaluated values.

#### Variables and Property Access

```html
<!-- Dot notation -->
${properties.jcr:title}
${currentPage.title}

<!-- Bracket notation (required for special characters) -->
${properties['jcr:title']}
${properties['cq:tags']}

<!-- Nested access -->
${currentPage.parent.title}
```

#### Literals

```html
<!-- Strings (single or double quotes) -->
${'Hello World'}
${"Hello World"}

<!-- Integers and floats -->
${42}
${3.14}

<!-- Booleans -->
${true}
${false}

<!-- Array literals -->
${[1, 2, 3]}
${['red', 'green', 'blue']}
```

#### Operators

```html
<!-- Logical operators -->
${properties.showTitle && properties.jcr:title}
${properties.title || 'Default Title'}
${!wcmmode.edit}

<!-- Comparison operators -->
${currentPage.depth == 3}
${properties.count != 0}
${properties.rating > 3}
${properties.index >= 0}

<!-- Ternary operator (colon MUST have surrounding spaces) -->
${wcmmode.edit ? 'Edit Mode' : 'View Mode'}
${properties.title ? properties.title : 'Untitled'}

<!-- 'in' operator (checks containment in string, array, or object) -->
${'foo' in myArray}
${'key' in myObject}
```

#### String Concatenation and Options

```html
<!-- String concatenation (NO + operator — use format or join) -->
${'Page: ' @ format=currentPage.title}

<!-- Join arrays -->
${myArray @ join=', '}

<!-- Internationalization -->
${'Search' @ i18n}
${'Search' @ i18n, locale='fr'}
${'Found {0} results' @ i18n, format=[resultCount]}

<!-- URI manipulation -->
${'path/page' @ extension='html'}
${'path/page' @ selectors='mobile', extension='json'}
${'path/page' @ prependPath='/content/site'}
${'path/page' @ appendPath='jcr:content'}
```

### Block Statements

All HTL block statements use HTML5 `data-sly-*` data attributes.

#### data-sly-use — Load Resources and Models

```html
<!-- Load a Sling Model (preferred) -->
<sly data-sly-use.model="com.myproject.core.models.HeroModel"/>
<h1>${model.title}</h1>

<!-- Load a Sling Model by resourceType lookup -->
<sly data-sly-use.model="com.myproject.core.models.HeroModel"/>

<!-- Load an HTL template library -->
<sly data-sly-use.tmpl="core/wcm/components/commons/v1/templates.html"/>

<!-- Load clientlib template -->
<sly data-sly-use.clientlib="/libs/granite/sightly/templates/clientlib.html"/>

<!-- Pass parameters to Sling Model via RequestAttributes -->
<sly data-sly-use.model="${'com.myproject.core.models.Card' @
    title=item.title,
    description=item.description}"/>
```

#### data-sly-test — Conditional Rendering

```html
<!-- Simple conditional -->
<h1 data-sly-test="${properties.jcr:title}">${properties.jcr:title}</h1>

<!-- Store test result in a variable for reuse -->
<sly data-sly-test.hasTitle="${properties.jcr:title}"/>
<h1 data-sly-test="${hasTitle}">${properties.jcr:title}</h1>
<p data-sly-test="${!hasTitle}">No title set</p>

<!-- Combine conditions -->
<div data-sly-test="${properties.showHero && properties.heroImage}">
    <!-- Hero content -->
</div>

<!-- Check WCM mode -->
<div data-sly-test="${wcmmode.edit}">
    <p>Visible only in edit mode</p>
</div>
```

#### data-sly-list — Iterate Arrays (Content-Only)

`data-sly-list` repeats the element content but removes the host element.

```html
<!-- Basic list iteration -->
<ul>
    <li data-sly-list="${currentPage.listChildren}">${item.title}</li>
</ul>

<!-- Named iterator with index -->
<ul data-sly-list.page="${currentPage.listChildren}">
    <li>${pageList.index}: ${page.title}</li>
</ul>

<!-- Available implicit variables (default 'item' and 'itemList'): -->
<!-- itemList.index (0-based), itemList.count (1-based) -->
<!-- itemList.first, itemList.middle, itemList.last -->
<!-- itemList.odd, itemList.even -->
```

#### data-sly-repeat — Iterate Arrays (Host Element Repeated)

`data-sly-repeat` repeats the host element itself for each item.

```html
<!-- Repeat the div for each item -->
<div data-sly-repeat="${model.items}" class="card">
    <h3>${item.title}</h3>
    <p>${item.description}</p>
</div>

<!-- Named repeat with metadata -->
<option data-sly-repeat.color="${model.colors}"
        value="${color}"
        selected="${colorList.first}">
    ${color}
</option>
```

#### data-sly-include — Include Another HTL Script

```html
<!-- Include a partial -->
<sly data-sly-include="partials/header.html"/>

<!-- Include with WCM mode override -->
<sly data-sly-include="${'partials/readonly.html' @ wcmmode='disabled'}"/>
```

#### data-sly-resource — Include Another AEM Component

```html
<!-- Include a resource by relative path -->
<div data-sly-resource="content/header"></div>

<!-- Force a specific resourceType -->
<sly data-sly-resource="${'hero' @ resourceType='myproject/components/hero'}"/>

<!-- Manipulate selectors -->
<sly data-sly-resource="${'content' @
    resourceType='myproject/components/text',
    selectors='mobile',
    addSelectors='compact',
    removeSelectors='full',
    replaceSelectors='mobile.compact'}"/>

<!-- Decoration tag and CSS class -->
<sly data-sly-resource="${'parsys' @
    resourceType='wcm/foundation/components/responsivegrid',
    decorationTagName='section',
    cssClassName='container'}"/>
```

#### data-sly-template / data-sly-call — Reusable Fragments

```html
<!-- Define a template -->
<template data-sly-template.card="${@ title, description, link}">
    <div class="card">
        <h3>${title}</h3>
        <p>${description}</p>
        <a href="${link}" class="card__cta">Learn More</a>
    </div>
</template>

<!-- Call a template -->
<sly data-sly-call="${card @
    title='My Title',
    description='My description',
    link='/content/mysite/page'}"/>

<!-- Define template library in a separate file (templates/card.html) -->
<sly data-sly-use.cardTmpl="templates/card.html"/>
<sly data-sly-call="${cardTmpl.card @
    title=properties.jcr:title,
    description=properties.jcr:description,
    link=properties.linkURL}"/>
```

#### data-sly-unwrap — Remove Host Element

```html
<!-- Render children without the wrapping div -->
<div data-sly-unwrap>
    <p>Only this paragraph is rendered</p>
</div>

<!-- Conditional unwrap -->
<div data-sly-unwrap="${!wcmmode.edit}" class="author-helper">
    <p>Wrapper shown only in edit mode</p>
</div>
```

#### data-sly-element — Dynamic Element Name

```html
<!-- Dynamic heading level -->
<h2 data-sly-element="${properties.headingLevel || 'h2'}">
    ${properties.jcr:title}
</h2>
```

#### data-sly-attribute — Dynamic Attributes

```html
<!-- Single attribute -->
<a data-sly-attribute.href="${properties.linkURL}">Link</a>

<!-- Multiple attributes via map -->
<input data-sly-attribute="${{
    type: 'text',
    name: properties.fieldName,
    placeholder: properties.placeholder,
    required: properties.required
}}"/>

<!-- Boolean attributes (false/null removes the attribute) -->
<input data-sly-attribute.disabled="${properties.isDisabled}"/>

<!-- Dynamic class (empty values are automatically removed) -->
<div data-sly-attribute.class="${model.cssClasses}">Content</div>
```

#### data-sly-set — Create Variables

```html
<!-- Store a computed value for reuse -->
<sly data-sly-set.fullName="${properties.firstName} ${properties.lastName}"/>
<h1>${fullName}</h1>

<!-- Store a conditional for multiple uses -->
<sly data-sly-set.isPublished="${properties.published && !properties.hidden}"/>
```

#### data-sly-text — Replace Element Text Content

```html
<!-- Replace text content (auto-escaped) -->
<p data-sly-text="${properties.description}">Placeholder text</p>
```

### The sly Element

When no existing HTML element can host a block statement, use `<sly>` — it is automatically removed from the output.

```html
<!-- CORRECT: sly element for logic-only blocks -->
<sly data-sly-test="${properties.showBanner}">
    <div class="banner">${properties.bannerText}</div>
</sly>

<!-- CORRECT: sly for includes and templates -->
<sly data-sly-use.clientlib="/libs/granite/sightly/templates/clientlib.html"
     data-sly-call="${clientlib.all @ categories='myproject.base'}"/>

<!-- INCORRECT: do not use sly when an existing element works -->
<sly data-sly-test="${properties.title}">
    <h1>${properties.title}</h1>
</sly>
<!-- CORRECT: attach to the element directly -->
<h1 data-sly-test="${properties.title}">${properties.title}</h1>
```

### Context-Aware XSS Escaping

HTL automatically applies context-aware escaping based on expression position. Override with `@context` only when automatic detection is insufficient.

#### Display Contexts

| Context | Purpose | Example |
|---------|---------|---------|
| `html` | Filters HTML, removes dangerous tags | `${richText @ context='html'}` |
| `text` | Encodes all HTML special characters (default for text nodes) | `${title @ context='text'}` |
| `attribute` | Encodes HTML special characters for attributes | `${value @ context='attribute'}` |
| `uri` | Validates URI, outputs nothing if invalid (default for href/src) | `${link @ context='uri'}` |
| `number` | Validates as number, outputs nothing if invalid | `${count @ context='number'}` |
| `attributeName` | Validates attribute name when dynamically setting attribute names | `${attrName @ context='attributeName'}` |
| `elementName` | Validates element name (allows safe HTML elements only) | `${tag @ context='elementName'}` |
| `scriptToken` | Validates JavaScript identifiers and literals | `${varName @ context='scriptToken'}` |
| `scriptString` | Encodes characters that would break out of JS strings | `${text @ context='scriptString'}` |
| `scriptComment` | Validates JavaScript comment safety | `${text @ context='scriptComment'}` |
| `styleToken` | Validates CSS identifiers, numbers, colors, dimensions | `${prop @ context='styleToken'}` |
| `styleString` | Encodes characters that would break out of CSS strings | `${fontName @ context='styleString'}` |
| `styleComment` | Validates CSS comment safety | `${text @ context='styleComment'}` |
| `unsafe` | Disables all escaping completely | `${markup @ context='unsafe'}` |

#### Automatic Context Detection

```html
<!-- HTL detects context automatically for most positions -->
<p>${properties.text}</p>                        <!-- text context -->
<a href="${properties.url}">Link</a>              <!-- uri context -->
<div class="${properties.cssClass}">Content</div> <!-- attribute context -->

<!-- You MUST specify context for script and style blocks -->
<script>
    var trackingId = "${model.trackingId @ context='scriptString'}";
    var config = ${model.configJson @ context='unsafe'};
</script>

<style>
    .component { font-family: "${model.fontFamily @ context='styleString'}"; }
</style>
```

#### Security Rules

```html
<!-- NEVER use unsafe context for user-supplied data -->
<!-- WRONG -->
${userInput @ context='unsafe'}

<!-- CORRECT — use 'html' for rich text from RTE -->
${properties.richText @ context='html'}

<!-- HTL removes output in script/style if context not declared -->
<!-- WRONG — will output nothing -->
<script>var x = "${properties.value}";</script>

<!-- CORRECT — explicit context required -->
<script>var x = "${properties.value @ context='scriptString'}";</script>

<!-- Safe pattern for inline JSON data -->
<div data-config="${model.jsonConfig @ context='attribute'}">Content</div>
```

### Use-API Patterns

#### Sling Models (Preferred)

Sling Models are the recommended approach for all HTL backend logic.

```html
<sly data-sly-use.model="com.myproject.core.models.HeroComponent"/>
<div class="hero" data-sly-test="${!model.empty}">
    <h1>${model.title}</h1>
    <p>${model.description}</p>
</div>
```

```java
@Model(
    adaptables = SlingHttpServletRequest.class,
    adapters = HeroComponent.class,
    resourceType = "myproject/components/hero",
    defaultInjectionStrategy = DefaultInjectionStrategy.OPTIONAL
)
public class HeroComponentImpl implements HeroComponent {

    @ValueMapValue
    private String title;

    @ValueMapValue
    private String description;

    @Override
    public String getTitle() { return title; }

    @Override
    public String getDescription() { return description; }

    @Override
    public boolean isEmpty() {
        return StringUtils.isBlank(title);
    }
}
```

#### Java Use-API (WCMUsePojo) — Legacy, Avoid in New Code

```java
// AVOID: WCMUsePojo is legacy — use Sling Models instead
public class MyHelper extends WCMUsePojo {
    private String title;

    @Override
    public void activate() throws Exception {
        title = getProperties().get("jcr:title", String.class);
    }

    public String getTitle() { return title; }
}
```

```html
<sly data-sly-use.helper="com.myproject.core.helpers.MyHelper"/>
<h1>${helper.title}</h1>
```

#### JavaScript Use-API — Deprecated

The JavaScript Use-API is **deprecated** for AEM as a Cloud Service. Do not use it in new projects.

### HTL Global Objects

#### Enumerable Objects (support dot notation and iteration)

| Object | Description |
|--------|-------------|
| `properties` | Current resource's ValueMap |
| `pageProperties` | Current page's properties |
| `inheritedPageProperties` | Inherited page properties up the hierarchy |

#### Java-Backed Objects

| Object | Description |
|--------|-------------|
| `currentPage` | `com.day.cq.wcm.api.Page` — current page |
| `resourcePage` | `com.day.cq.wcm.api.Page` — page containing the resource |
| `currentNode` | `javax.jcr.Node` — current JCR node |
| `resource` | `org.apache.sling.api.resource.Resource` — current resource |
| `resolver` | `org.apache.sling.api.resource.ResourceResolver` |
| `request` | `org.apache.sling.api.SlingHttpServletRequest` |
| `response` | `org.apache.sling.api.SlingHttpServletResponse` |
| `component` | `com.day.cq.wcm.api.components.Component` — current component |
| `componentContext` | `com.day.cq.wcm.api.components.ComponentContext` |
| `wcmmode` | WCM mode object with `.edit`, `.preview`, `.design`, `.disabled` booleans |
| `pageManager` | `com.day.cq.wcm.api.PageManager` |
| `currentDesign` | Design of the current page |
| `currentStyle` | Style of the current resource |
| `currentContentPolicy` | Content policy for the current component |
| `currentContentPolicyProperties` | Properties of the current content policy |
| `xssAPI` | XSS protection utilities |
| `log` | SLF4J Logger |

### Common Patterns

#### Loading Client Libraries

```html
<sly data-sly-use.clientlib="/libs/granite/sightly/templates/clientlib.html">
    <sly data-sly-call="${clientlib.css @ categories='myproject.base'}"/>
</sly>

<!-- In page footer -->
<sly data-sly-use.clientlib="/libs/granite/sightly/templates/clientlib.html">
    <sly data-sly-call="${clientlib.js @ categories='myproject.base'}"/>
</sly>

<!-- CSS and JS together -->
<sly data-sly-use.clientlib="/libs/granite/sightly/templates/clientlib.html"
     data-sly-call="${clientlib.all @ categories='myproject.base'}"/>
```

#### Placeholder Pattern for Authors

```html
<sly data-sly-use.model="com.myproject.core.models.TextComponent"/>
<div class="cmp-text" data-sly-test.hasContent="${!model.empty}">
    <div class="cmp-text__content" data-sly-text="${model.text @ context='html'}"></div>
</div>
<sly data-sly-test="${!hasContent && wcmmode.edit}">
    <div class="cq-placeholder" data-emptytext="${component.title}"></div>
</sly>
```

#### Responsive Image with Data Attributes

```html
<sly data-sly-use.model="com.myproject.core.models.ImageComponent"/>
<div data-sly-test="${model.src}" class="cmp-image"
     data-sly-attribute="${{
         'data-src': model.src,
         'data-widths': model.widths @ join=','
     }}">
    <img src="${model.src}" alt="${model.alt}"/>
</div>
```

#### Iterating Child Pages for Navigation

```html
<nav data-sly-test="${currentPage.listChildren}">
    <ul data-sly-list.child="${currentPage.listChildren}">
        <li class="${child.name == request.requestPathInfo.selectorString ? 'active' : ''}">
            <a href="${child.path}.html">${child.title}</a>
        </li>
    </ul>
</nav>
```

### Performance Guidelines

- **Avoid excessive data-sly-resource nesting**: Each `data-sly-resource` triggers a new Sling resolution cycle. Deeply nested includes degrade performance.
- **Use data-sly-test early**: Skip entire blocks of markup when conditions are false rather than rendering hidden content.
- **Prefer Sling Models over WCMUsePojo**: Sling Models are cached and use injection rather than manual property lookups.
- **Minimize logic in HTL**: Push complex computations, loops, and data transformations to Sling Models. HTL should focus on rendering.
- **Reuse templates with data-sly-template/call**: Extract repeated markup patterns into template libraries instead of duplicating HTL code.
- **Avoid calling Java methods with side effects**: HTL getter access should be idempotent since the template engine may cache or reorder calls.

### Anti-Patterns to Avoid

```html
<!-- WRONG: Business logic in HTL -->
<sly data-sly-list.page="${currentPage.listChildren}">
    <sly data-sly-test="${page.properties.hideInNav != 'true'
        && page.properties['cq:lastModified']}">
        <!-- complex filtering belongs in a Sling Model -->
    </sly>
</sly>

<!-- CORRECT: Delegate to Sling Model -->
<sly data-sly-use.nav="com.myproject.core.models.Navigation"/>
<sly data-sly-list.page="${nav.visiblePages}">
    <li>${page.title}</li>
</sly>

<!-- WRONG: Using context='unsafe' for user content -->
<div>${properties.description @ context='unsafe'}</div>

<!-- CORRECT: Use 'html' for rich text -->
<div>${properties.description @ context='html'}</div>

<!-- WRONG: Deeply nested data-sly-resource calls -->
<sly data-sly-resource="${'a' @ resourceType='comp/a'}">
    <!-- comp/a includes comp/b which includes comp/c which includes comp/d -->
</sly>

<!-- WRONG: Multiple block statements of the same type on one element -->
<!-- Only the first data-sly-test is evaluated -->
<div data-sly-test="${condA}" data-sly-test="${condB}">Never do this</div>

<!-- CORRECT: Combine conditions -->
<div data-sly-test="${condA && condB}">Combined condition</div>
```
