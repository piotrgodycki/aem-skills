---
title: Edge Delivery Services Forms and Spreadsheet Data
impact: HIGH
impactDescription: Forms and spreadsheet-driven content are core EDS patterns — correct implementation ensures data integrity, performance, and usability
tags: edge-delivery, eds, forms, spreadsheets, json, adaptive-forms, data-sources, google-sheets, excel
---

## EDS Forms and Spreadsheet Data

Edge Delivery Services provides two primary data patterns: the Adaptive Forms Block for capturing user input, and spreadsheet-to-JSON feeds for powering dynamic block content. Both patterns use Google Sheets or Microsoft Excel/SharePoint as the underlying data store.

### Forms in EDS: Two Authoring Approaches

1. **Document-based authoring** — Define forms in spreadsheets (Excel or Google Sheets) for simple forms with basic fields
2. **WYSIWYG authoring** — Use the Universal Editor for complex forms needing panels, business logic, and AEM Workflows

### Form Block Pattern (Document-Based)

The Adaptive Forms Block is a table-based placeholder that embeds a form defined in a spreadsheet.

#### Step 1: Create a Form Spreadsheet

Create a spreadsheet (e.g., `contact-us`) in your project folder. The form definition goes in a sheet named `shared-aem` (or `helix-default`, deprecated).

Each row is a form field. Column headers define field properties:

| Type     | Name        | Label           | Placeholder          | Mandatory | Value   |
|----------|-------------|-----------------|----------------------|-----------|---------|
| text     | first-name  | First Name      | Enter first name     | true      |         |
| text     | last-name   | Last Name       | Enter last name      | true      |         |
| email    | email       | Email Address   | you@example.com      | true      |         |
| tel      | phone       | Phone Number    | +1 (555) 000-0000    | false     |         |
| select   | department  | Department      |                      | true      |         |
| textarea | message     | Your Message    | How can we help?     | false     |         |
| submit   | submit      | Send Message    |                      |           |         |

#### Step 2: Embed in a Document

In your page document, create a one-column, two-row table:

| Form                                                      |
|-----------------------------------------------------------|
| https://main--mysite--owner.aem.live/contact-us.json      |

The second row contains the preview URL to the form spreadsheet JSON endpoint.

### Form Field Types and Properties

All valid HTML5 input types are supported:

| Type             | Use Case                              |
|------------------|---------------------------------------|
| `text`           | Single-line text input                |
| `email`          | Email with built-in validation        |
| `tel`            | Phone number input                    |
| `number`         | Numeric input with min/max            |
| `date`           | Date picker                           |
| `textarea`       | Multi-line text                       |
| `select`         | Dropdown menu                         |
| `checkbox`       | Single or grouped checkboxes          |
| `radio`          | Radio button group                    |
| `file`           | File upload                           |
| `hidden`         | Hidden field                          |
| `submit`         | Submit button                         |
| `reset`          | Reset button                          |
| `fieldset`       | Groups related fields (panel)         |

#### Core Field Properties

| Property             | Scope              | Description                                    |
|----------------------|--------------------|------------------------------------------------|
| Type                 | All                | HTML5 input type or container type              |
| Name                 | All                | Field identifier for submission                 |
| Label                | All                | Display label text                              |
| Value                | Input, select      | Default/initial value                           |
| Placeholder          | Text inputs        | Hint text for expected input                    |
| Description          | All                | Additional help text below the field            |
| Visible              | All                | Boolean for initial visibility                  |
| Mandatory            | Most inputs        | Whether field must be filled before submission  |
| Min / Max            | Number, date, range| Acceptable value boundaries                     |
| Accept               | File inputs        | Allowed MIME types (e.g., `.pdf,.jpg`)          |
| Multiple             | File, select       | Allow multiple selections or files              |
| Options              | Select, radio      | Comma-separated list of choices                 |
| Checked              | Checkbox, radio    | Default selected state                          |
| Fieldset             | All                | Groups fields into a fieldset with legend        |
| Repeatable           | Fieldset           | Allow fieldset repetition (with Min/Max count)  |
| Visible Expression   | All                | Spreadsheet formula for dynamic visibility       |
| Value Expression     | All                | Spreadsheet formula for dynamic field values     |

### Form Validation

EDS forms support both HTML5 native validation and expression-based rules:

```
# In the spreadsheet, use the Mandatory column:
Mandatory: true           → required field
Mandatory: false          → optional field

# For pattern validation, use the Type column:
Type: email               → built-in email validation
Type: tel                 → phone format

# For min/max constraints:
Min: 1, Max: 100          → numeric range
Min: 2025-01-01           → minimum date
```

Dynamic visibility with spreadsheet formulas in the `Visible Expression` column:

```
=IF(department="Sales", true, false)
```

### Form Submission Handling

#### Spreadsheet Submission (Default)

Forms submit data to an `incoming` sheet in the same workbook. Setup:

1. Add a sheet named `incoming` to the form workbook
2. Create a table named `intake_form` with column headers matching form field names (excluding the submit button)
3. Preview the form via Sidekick

Or use the Admin API to auto-generate the incoming sheet:

```bash
curl -X POST \
  'https://admin.aem.page/form/{owner}/{repo}/{branch}/contact-us.json' \
  -H 'Content-Type: application/json' \
  -d '{"data": {"first-name": "John", "last-name": "Doe", "email": "john@example.com"}}'
```

The system auto-creates a `Slack` sheet for notification configuration (specify `teamId` and channel).

#### Custom Endpoint Submission

Override the default submission target with an `action` property in the form definition:

| Type   | Name   | Label  | Action                           |
|--------|--------|--------|----------------------------------|
| submit | submit | Submit | https://api.example.com/submit   |

#### Forms Submission Service

The Forms Submission Service stores data in OneDrive, SharePoint, or Google Sheets, plus supports:
- Azure Blob Storage
- REST endpoints
- Salesforce, Microsoft Dynamics OData
- Marketo Engage, Adobe Experience Platform
- AEM Workflows, Power Automate flows
- Email addresses

### reCAPTCHA / Bot Protection

Google reCAPTCHA is supported for WYSIWYG-authored forms. For document-based forms, implement custom protection:

```javascript
// In a custom form handler, add honeypot field pattern
export default function decorate(block) {
  const form = block.querySelector('form');
  if (!form) return;

  // Add a honeypot field (hidden from real users)
  const honeypot = document.createElement('input');
  honeypot.type = 'text';
  honeypot.name = 'website_url';
  honeypot.style.display = 'none';
  honeypot.tabIndex = -1;
  honeypot.autocomplete = 'off';
  form.prepend(honeypot);

  form.addEventListener('submit', (e) => {
    if (honeypot.value) {
      e.preventDefault(); // Bot detected
      return;
    }
    // Continue with normal submission
  });
}
```

### Multi-Step Forms

Create multi-step forms using fieldsets as panels. Use the `Fieldset` property to group fields, then show/hide panels with JavaScript:

```javascript
export default function decorate(block) {
  const form = block.querySelector('form');
  const fieldsets = form.querySelectorAll('fieldset');
  let currentStep = 0;

  // Hide all fieldsets except the first
  fieldsets.forEach((fs, i) => {
    fs.style.display = i === 0 ? 'block' : 'none';
  });

  // Add navigation buttons
  const nav = document.createElement('div');
  nav.className = 'form-nav';
  nav.innerHTML = `
    <button type="button" class="prev" disabled>Previous</button>
    <button type="button" class="next">Next</button>
  `;
  form.appendChild(nav);

  nav.querySelector('.next').addEventListener('click', () => {
    if (currentStep < fieldsets.length - 1) {
      fieldsets[currentStep].style.display = 'none';
      currentStep += 1;
      fieldsets[currentStep].style.display = 'block';
    }
  });

  nav.querySelector('.prev').addEventListener('click', () => {
    if (currentStep > 0) {
      fieldsets[currentStep].style.display = 'none';
      currentStep -= 1;
      fieldsets[currentStep].style.display = 'block';
    }
  });
}
```

### Custom Form Field Types

Extend the form block by overriding field rendering:

```javascript
// blocks/form/custom-fields.js
export function decorateRating(fieldWrapper) {
  const input = fieldWrapper.querySelector('input');
  const stars = document.createElement('div');
  stars.className = 'rating-stars';
  for (let i = 1; i <= 5; i += 1) {
    const star = document.createElement('span');
    star.textContent = '\u2605';
    star.dataset.value = i;
    star.addEventListener('click', () => {
      input.value = i;
      stars.querySelectorAll('span').forEach((s) => {
        s.classList.toggle('active', Number(s.dataset.value) <= i);
      });
    });
    stars.appendChild(star);
  }
  input.style.display = 'none';
  fieldWrapper.appendChild(stars);
}
```

---

## Spreadsheet Data for Dynamic Block Content

Beyond forms, spreadsheets serve as data sources for any block that needs structured content (pricing tables, team lists, event calendars, FAQ lists, etc.).

### How Spreadsheets Become JSON Endpoints

Any spreadsheet in your project folder is automatically available as a JSON endpoint after preview/publish:

```
https://main--mysite--owner.aem.live/data/pricing.json
```

### Sheet Naming Conventions

| Convention               | Behavior                                           |
|--------------------------|----------------------------------------------------|
| Single sheet workbook    | Automatically used as the data source              |
| `shared-default`         | Default sheet in multi-sheet workbooks              |
| `shared-<name>`          | Exposed as a named sheet; others remain hidden      |
| `helix-default`          | Deprecated; use `shared-default` instead            |
| `helix-<name>`           | Deprecated; use `shared-<name>` instead             |

Only sheets prefixed with `shared-` are delivered publicly. Other sheets remain hidden.

### JSON Response Format (Single Sheet)

```json
{
  "total": 42,
  "offset": 0,
  "limit": 42,
  "data": [
    { "name": "Basic", "price": "$9/mo", "features": "5 users, 10GB" },
    { "name": "Pro", "price": "$29/mo", "features": "25 users, 100GB" }
  ],
  ":colWidths": [200, 100, 300],
  ":type": "sheet"
}
```

### Multi-Sheet Format

When a workbook has multiple `shared-` sheets, the response is:

```json
{
  ":names": ["plans", "features"],
  "plans": { "total": 3, "offset": 0, "limit": 3, "data": [...] },
  "features": { "total": 10, "offset": 0, "limit": 10, "data": [...] },
  ":type": "multi-sheet",
  ":version": 3
}
```

### Query Parameters for Spreadsheet Data

| Parameter     | Example                              | Description                          |
|---------------|--------------------------------------|--------------------------------------|
| `sheet`       | `?sheet=plans`                       | Select specific sheet (`shared-plans`) |
| Multiple      | `?sheet=plans&sheet=features`        | Return multiple named sheets          |
| `limit`       | `?limit=10`                          | Limit rows returned (default: 1000)   |
| `offset`      | `?offset=20`                         | Skip rows for pagination              |

### Fetching Spreadsheet Data in Blocks

```javascript
export default async function decorate(block) {
  // Fetch pricing data from spreadsheet
  const resp = await fetch('/data/pricing.json');
  if (!resp.ok) return;

  const { data } = await resp.json();

  // Build pricing cards
  const grid = document.createElement('div');
  grid.className = 'pricing-grid';

  data.forEach((plan) => {
    const card = document.createElement('div');
    card.className = 'pricing-card';
    card.innerHTML = `
      <h3>${plan.name}</h3>
      <p class="price">${plan.price}</p>
      <p class="features">${plan.features}</p>
      <a href="${plan.cta_link}" class="button primary">${plan.cta_text}</a>
    `;
    grid.appendChild(card);
  });

  block.textContent = '';
  block.appendChild(grid);
}
```

### Handling Array Values from Spreadsheets

Cell values that look like arrays are delivered as strings. Parse them with `JSON.parse()`:

```javascript
const tags = JSON.parse(item.tags); // '["Adobe Life","Responsibility"]' → array
```

### Paginated Fetching for Large Datasets

```javascript
async function fetchAllData(path) {
  const pageSize = 100;
  let offset = 0;
  let allData = [];
  let hasMore = true;

  while (hasMore) {
    const resp = await fetch(`${path}?limit=${pageSize}&offset=${offset}`);
    if (!resp.ok) break;
    const json = await resp.json();
    allData = allData.concat(json.data);
    offset += pageSize;
    hasMore = json.data.length === pageSize && offset < json.total;
  }

  return allData;
}
```

### Google Sheets vs Excel/SharePoint as Data Source

Both platforms work identically from the delivery perspective:

| Feature                  | Google Sheets                   | Excel/SharePoint                |
|--------------------------|---------------------------------|---------------------------------|
| Share permissions         | Share with AEM service account  | Share with AEM service account  |
| Sheet naming              | `shared-` prefix                | `shared-` prefix                |
| JSON endpoint             | Same structure                  | Same structure                  |
| Real-time collaboration   | Native                          | Native                          |
| File extension            | Auto-detected                   | `.xlsx`                         |

### Caching and Refresh Patterns

Spreadsheet JSON endpoints are cached at the CDN. Data updates follow the standard EDS publish cycle:

1. Edit the spreadsheet
2. Preview via Sidekick (updates preview CDN)
3. Publish via Sidekick (updates production CDN)

For blocks that fetch spreadsheet data at runtime:

```javascript
// Use cache-busting only in preview, never in production
const isPreview = window.location.hostname.includes('.aem.page');
const cacheBuster = isPreview ? `?_=${Date.now()}` : '';
const resp = await fetch(`/data/pricing.json${cacheBuster}`);
```

CDN caching headers are managed by EDS automatically. Do not set custom `Cache-Control` headers for spreadsheet endpoints.

### Dynamic Block Content Examples

#### Team List from Spreadsheet

Spreadsheet columns: `name`, `role`, `image`, `bio`, `linkedin`

```javascript
export default async function decorate(block) {
  const resp = await fetch('/data/team.json');
  if (!resp.ok) return;
  const { data } = await resp.json();

  const list = document.createElement('ul');
  list.className = 'team-list';

  data.forEach((member) => {
    const li = document.createElement('li');
    li.className = 'team-member';
    li.innerHTML = `
      <img src="${member.image}" alt="${member.name}" loading="lazy" width="200" height="200">
      <h3>${member.name}</h3>
      <p class="role">${member.role}</p>
      <p class="bio">${member.bio}</p>
      ${member.linkedin ? `<a href="${member.linkedin}" target="_blank" rel="noopener">LinkedIn</a>` : ''}
    `;
    list.appendChild(li);
  });

  block.textContent = '';
  block.appendChild(list);
}
```

### Anti-Patterns

- **Client-side-only validation** — Always pair client-side validation with server-side checks; HTML5 validation can be bypassed by disabling JavaScript
- **Large spreadsheet fetches** — Never fetch entire large spreadsheets at once; use `limit` and `offset` for pagination; default response cap is 1000 rows
- **No error handling** — Always check `resp.ok` and provide fallback UI when spreadsheet data is unavailable
- **Storing PII in shared spreadsheets** — Form submission spreadsheets with the `incoming` sheet are accessible; never store sensitive personal data there
- **Ignoring the `shared-` prefix** — Sheets without the `shared-` prefix are not delivered; using the deprecated `helix-` prefix still works but should be migrated
- **Fetching data on every page load without need** — If data rarely changes, consider rendering content statically in the document rather than fetching from a spreadsheet
- **Not previewing/publishing spreadsheet changes** — Edits to spreadsheets are not reflected until you preview and publish via the Sidekick
- **Using custom Cache-Control headers** — EDS manages CDN caching; overriding headers causes stale data or cache invalidation issues
- **Missing loading states** — Blocks that fetch data asynchronously should show a loading indicator or skeleton to avoid layout shift
