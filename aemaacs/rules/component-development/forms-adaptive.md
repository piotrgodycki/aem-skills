---
title: AEM Forms - Adaptive Forms Frontend
impact: HIGH
impactDescription: Incorrect form implementation causes broken submissions, poor accessibility, and failed integrations
tags: forms, adaptive-forms, core-components, form-theming, rule-editor, captcha, headless-forms, form-data-model, accessibility
---

## AEM Forms (Adaptive Forms) Frontend

Adaptive Forms Core Components provide 30+ open-source, BEM-compliant form components built on AEM WCM Core Components. They support responsive rendering across devices, theming via SCSS, a visual rule editor for dynamic behavior, and multiple submission backends including REST, SharePoint, and OneDrive.

### Adaptive Forms Core Components (v3)

The current generation uses Core Components v3 with automatic upgrade support. Enable them via the Forms Container on AEM Sites pages or as standalone forms.

#### Form Container

The Form Container is the root component wrapping all form fields. It defines:
- **Form model:** None, JSON Schema, or Form Data Model (FDM)
- **Submit action:** REST endpoint, email, SharePoint, OneDrive, Power Automate, custom
- **Prefill service:** FDM-based, custom OSGi service, or URL-parameter-based
- **Title and metadata**
- **Redirect/thank-you page after submission**

```html
<!-- HTL: Embedding a form container in Sites page -->
<div data-sly-resource="${'myform' @ resourceType='forms/components/form/container'}"
     class="cmp-form-container">
</div>
```

#### Form Field Components

| Component | Purpose | Key Properties |
|-----------|---------|---------------|
| **Text Input** | Single-line text | placeholder, pattern, maxLength, required |
| **Email** | Email with validation | pattern, multiple |
| **Numeric Input** | Numbers with min/max | minimum, maximum, step |
| **Date Picker** | Date selection | minDate, maxDate, displayFormat |
| **Dropdown** | Single/multi-select | options (static or dynamic), multiSelect |
| **Checkbox Group** | Multiple selections | options, minItems, maxItems |
| **Radio Button** | Single selection | options, layout (horizontal/vertical) |
| **File Attachment** | File upload | accept, maxFileSize, minFiles, maxFiles |
| **Panel** | Grouping container | layout (responsive grid), collapsible |
| **Wizard** | Multi-step form | navigable steps, validation per step |
| **Horizontal Tabs** | Tabbed layout | tabs with independent validation |
| **Accordion** | Collapsible sections | expandAll, singleExpansion |
| **Text (Rich Text)** | Static content | richText, markdown support |
| **Signature** | Digital signature | scribble-based capture |
| **Submit Button** | Form submission | submitAction binding |

### Form Theming

Themes are SCSS-based packages deployed via the frontend pipeline. They follow BEM naming and can be customized at global or component level.

#### Theme Structure

```
mytheme/
├── package.json
├── package-lock.json
├── src/
│   ├── theme.scss                    # Root SCSS entry point
│   ├── site/
│   │   └── _site.scss                # Page-level styles
│   ├── resources/
│   │   ├── images/
│   │   └── fonts/
│   └── components/
│       ├── accordion/
│       │   └── _accordion.scss
│       ├── button/
│       │   └── _button.scss
│       ├── textinput/
│       │   └── _textinput.scss
│       ├── dropdown/
│       │   └── _dropdown.scss
│       └── _variables.scss           # Global variables (colors, fonts, spacing)
└── .env                              # AEM instance URL for live preview
```

#### Global Variables (_variables.scss)

```scss
// _variables.scss — modify these to change the entire theme
$font-family-primary: 'Adobe Clean', Helvetica, sans-serif;
$font-family-secondary: 'Adobe Clean Serif', Georgia, serif;
$color-primary: #1473e6;
$color-secondary: #2c2c2c;
$color-error: #d7373f;
$color-success: #268e6c;
$color-background: #ffffff;
$spacing-base: 8px;
$border-radius: 4px;
$input-border-color: #b3b3b3;
$input-focus-border-color: $color-primary;
```

#### Component-Level Styling (BEM)

```scss
// _textinput.scss — component-specific overrides
.cmp-adaptiveform-textinput {
  &__label {
    font-weight: 600;
    margin-bottom: $spacing-base / 2;
  }

  &__widget {
    border: 1px solid $input-border-color;
    border-radius: $border-radius;
    padding: $spacing-base;
    transition: border-color 0.2s ease;

    &:focus {
      border-color: $input-focus-border-color;
      outline: none;
      box-shadow: 0 0 0 2px rgba($color-primary, 0.2);
    }
  }

  &__errormessage {
    color: $color-error;
    font-size: 0.85rem;
    margin-top: $spacing-base / 4;
  }
}
```

**Priority rule:** When a style is defined at both theme and component level, the component-level style takes priority.

#### Pre-Built Themes

Adobe provides three reference themes (available on GitHub):
- **Canvas** -- minimal, clean baseline
- **WKND** -- example brand theme
- **EASEL** -- creative styling

#### Theme Development Workflow

```bash
# 1. Clone a reference theme
git clone https://github.com/adobe/aem-forms-theme-canvas.git mytheme

# 2. Update package.json with project name
# 3. Customize _variables.scss and component SCSS files

# 4. Live preview (requires .env with AEM URL)
npm run live

# 5. Build for deployment
npm run build

# 6. Deploy via frontend pipeline in Cloud Manager
```

### Form Rules and Expressions

The Rule Editor provides visual condition-action logic without custom code.

#### Rule Types

| Rule Type | Purpose | Example |
|-----------|---------|---------|
| **When** | If-then logic | When country = "US", show state dropdown |
| **Show** | Conditionally display | Show spouse fields when marital status = "Married" |
| **Hide** | Conditionally hide | Hide company fields for individual customers |
| **Enable** | Activate field | Enable submit when terms checkbox is checked |
| **Disable** | Deactivate field | Disable email when "same as primary" is checked |
| **Set Value Of** | Compute values | Set fullName = firstName + " " + lastName |
| **Validate** | Field validation | Validate age >= 18 for insurance form |
| **Invoke Service** | Call FDM backend | Fetch address details from ZIP code lookup |

#### Custom Functions

Extend the rule editor with JavaScript functions deployed as ClientLibs:

```javascript
/**
 * Calculate loan EMI
 * @name calculateEMI
 * @param {number} principal - Loan amount
 * @param {number} rate - Annual interest rate (%)
 * @param {number} tenure - Loan tenure in months
 * @returns {number} Monthly EMI
 */
function calculateEMI(principal, rate, tenure) {
  const monthlyRate = rate / 12 / 100;
  const emi = principal * monthlyRate * Math.pow(1 + monthlyRate, tenure)
              / (Math.pow(1 + monthlyRate, tenure) - 1);
  return Math.round(emi * 100) / 100;
}
```

Register custom functions via a ClientLib with category `customfunctionsdefinition`.

#### Permission Model

- **forms-power-users:** Create, edit, and delete rules/scripts
- **forms-users:** Can only use existing rules, cannot create new ones

### Form Submission Actions

| Action | Use Case | Configuration |
|--------|----------|---------------|
| **REST Endpoint** | Custom API backend | URL, method, headers, authentication |
| **Email** | Notification on submit | To, CC, subject, template |
| **Form Data Model** | Database/service write | FDM entity, write operation |
| **SharePoint** | Store in SharePoint list | Site, list, field mapping |
| **OneDrive** | Store as file in OneDrive | Folder path, file naming pattern |
| **Power Automate** | Trigger workflow | Flow URL, payload mapping |
| **AEM Workflow** | Start AEM workflow | Workflow model, payload path |

### Prefill Service Patterns

Prepopulate form fields from external data sources:

```json
// Prefill JSON structure matching form field binding
{
  "data": {
    "personalInfo": {
      "firstName": "John",
      "lastName": "Doe",
      "email": "john.doe@example.com"
    },
    "address": {
      "street": "123 Main St",
      "city": "San Jose",
      "state": "CA"
    }
  }
}
```

**Prefill sources:**
- **Form Data Model:** Uses the Get Service of the configured FDM to fetch data
- **Custom OSGi service:** Implement `com.adobe.forms.common.service.DataXMLProvider`
- **URL parameters:** Map query parameters to form fields
- **User profile:** Prefill from logged-in user attributes

### reCAPTCHA / CAPTCHA Integration

```
Supported CAPTCHA services:
- Google reCAPTCHA v2 (checkbox)
- Google reCAPTCHA Enterprise (score-based)
- hCaptcha
- Turnstile (Cloudflare)
```

Configure via Cloud Configuration at **Tools > Cloud Services > reCAPTCHA**, then add the CAPTCHA component to the form. Site key and secret key are stored as Cloud Service configurations.

### Custom Form Components

Build custom components by extending Core Components:

```
/apps/myproject/components/form/
└── custom-rating/
    ├── .content.xml          # sling:resourceSuperType = "core/fd/components/form/base/v1/base"
    ├── _cq_dialog/
    │   └── .content.xml      # Touch UI dialog
    ├── custom-rating.html    # HTL template
    └── clientlib/
        ├── js/
        │   └── rating.js     # Client-side behavior
        └── css/
            └── rating.css    # Component styling
```

### Form Fragments (Reusable Sections)

Create reusable form sections (address blocks, payment info) as fragments:
- Author fragments independently at **Forms > Forms & Documents**
- Reference fragments in multiple forms
- Changes to a fragment propagate to all forms using it
- Fragments support their own rules and validation
- Use Panel component as fragment container

### Form Data Model (FDM) and Backend Integration

FDM provides a unified data representation connecting forms to backend services:

```
Supported data sources:
- RESTful web services (Swagger/OpenAPI)
- SOAP web services (WSDL)
- OData services
- RDBMS (via JDBC - Cloud Service uses indirect connection via API)
- Microsoft Dynamics 365
- Salesforce
```

**FDM operations:**
- **Get:** Read data (prefill)
- **Create:** Write new records
- **Update:** Modify existing records
- **Delete:** Remove records

Configure at **Tools > Cloud Services > Data Sources**, then create FDM at **Tools > Forms > Data Integrations**.

### Client-Side Validation Patterns

```javascript
// Built-in validation types available on form fields
{
  "required": true,                    // Field is mandatory
  "pattern": "^[A-Z]{2}\\d{6}$",     // Regex pattern (e.g., passport number)
  "minLength": 3,                      // Minimum character count
  "maxLength": 50,                     // Maximum character count
  "minimum": 0,                        // Numeric minimum
  "maximum": 100,                      // Numeric maximum
  "exclusiveMinimum": true,           // Exclude boundary value
  "validateExpression": "field > 0",  // Custom expression
  "validationMessage": "Enter a valid passport number (e.g., AB123456)"
}
```

**Always pair client-side validation with server-side validation.** Client-side rules provide UX feedback; server-side validation ensures data integrity.

### Headless Adaptive Forms

Headless forms render from a JSON schema using any frontend framework:

```javascript
// Fetch form JSON definition
const response = await fetch(
  'https://publish-p12345-e67890.adobeaemcloud.com/content/forms/af/myform.model.json'
);
const formDefinition = await response.json();

// formDefinition contains:
// - Form structure (fields, panels, layout)
// - Validation rules
// - Submission configuration
// - Prefill data bindings
```

**Rendering options:**
- React SDK (`@aemforms/af-react-renderer`)
- Angular SDK
- Custom renderer using the JSON schema
- Mobile native apps (iOS/Android)

**Key capabilities preserved in headless mode:**
- Prefill via FDM
- Server-side validation
- Submit actions
- Workflow integration
- Document of Record generation

### Form Analytics and Tracking

- **Adobe Analytics integration:** Track form starts, abandonment, field time, submission
- **Adobe Experience Platform (AEP):** Stream form events to AEP for unified profiles
- **Custom events:** Use the AEM Client Data Layer to emit form interaction events

```javascript
// Listen for form events via the Client Data Layer
window.adobeDataLayer = window.adobeDataLayer || [];
window.adobeDataLayer.push(function(dl) {
  dl.addEventListener('cmp:show', function(event) {
    if (event.eventInfo.path.includes('adaptiveForm')) {
      // Track form view
    }
  });
});
```

### Accessibility in Forms (WCAG Compliance)

Adaptive Forms Core Components include built-in accessibility features:

- **ARIA labels:** Auto-generated `aria-label`, `aria-describedby`, `aria-required`
- **Keyboard navigation:** Full Tab/Shift+Tab navigation, Enter to submit
- **Screen reader support:** Field labels, help text, and error messages announced
- **Focus management:** Visible focus indicators, logical tab order
- **Color contrast:** Themes must meet WCAG 2.1 AA contrast ratios (4.5:1 text, 3:1 UI)
- **Error identification:** Errors linked to fields via `aria-describedby`

```html
<!-- Accessible form field output example -->
<div class="cmp-adaptiveform-textinput" data-cmp-is="adaptiveFormTextInput">
  <label for="firstName" class="cmp-adaptiveform-textinput__label">
    First Name <span class="cmp-adaptiveform-textinput__required">*</span>
  </label>
  <input id="firstName"
         type="text"
         aria-required="true"
         aria-describedby="firstName-help firstName-error"
         class="cmp-adaptiveform-textinput__widget" />
  <div id="firstName-help" class="cmp-adaptiveform-textinput__helptext">
    Enter your legal first name
  </div>
  <div id="firstName-error" class="cmp-adaptiveform-textinput__errormessage"
       role="alert" aria-live="polite"></div>
</div>
```

### Anti-Patterns

```
AVOID these common mistakes:

1. Client-only validation without server-side checks
   BAD:  Rely solely on JavaScript validation for required fields
   GOOD: Client-side validation for UX + server-side validation via submit action/FDM

2. Oversized single-page forms
   BAD:  50+ fields on a single panel with no progressive disclosure
   GOOD: Use Wizard or Tabs to break forms into logical steps (5-10 fields per step)

3. Missing error messages
   BAD:  Generic "Field is invalid" or no message at all
   GOOD: Specific, actionable messages: "Phone number must be 10 digits (e.g., 5551234567)"

4. Hardcoded form options instead of dynamic data
   BAD:  Static dropdown with country list embedded in dialog
   GOOD: FDM-backed dropdown pulling current country list from service

5. Ignoring mobile form UX
   BAD:  Date fields as text input on mobile
   GOOD: Use Date Picker component with native mobile date selection

6. Missing CAPTCHA on public forms
   BAD:  Public-facing form with no bot protection
   GOOD: reCAPTCHA Enterprise or hCaptcha on all anonymous submission forms

7. Loading entire form when only a fragment changed
   BAD:  Re-render full form on panel switch
   GOOD: Use lazy loading on Wizard/Tab panels to load content on demand

8. Custom components without accessibility
   BAD:  Custom rating widget with div+click handlers only
   GOOD: Custom component with role="radiogroup", aria-labels, keyboard support
```
