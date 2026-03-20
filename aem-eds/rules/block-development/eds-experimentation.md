---
title: Edge Delivery Services Experimentation & Personalization
impact: HIGH
impactDescription: Experimentation drives conversion optimization — correct implementation ensures valid results without performance degradation
tags: edge-delivery, eds, experimentation, a-b-testing, personalization, audiences, rum, analytics
---

## EDS Experimentation & Personalization

Edge Delivery Services includes a lightweight, privacy-first experimentation framework that enables A/B testing, audience targeting, and campaign personalization without slowing down your site. The framework is deeply integrated into the AEM delivery system and uses Real User Monitoring (RUM) for data collection.

### Experimentation Framework Overview

The `aem-experimentation` plugin provides three capabilities:
- **Experiments** — A/B testing with control and challenger pages
- **Audiences** — Content personalization based on visitor segments
- **Campaigns** — Targeted content delivery via URL parameters

Install the plugin as a git subtree:

```bash
git subtree add --squash --prefix plugins/experimentation \
  git@github.com:adobe/aem-experimentation.git main
```

Update later with:

```bash
git subtree pull --squash --prefix plugins/experimentation \
  git@github.com:adobe/aem-experimentation.git main
```

### Plugin Integration in scripts.js

Define audiences and register the plugin in your `scripts.js`:

```javascript
// Define audience resolver functions
const AUDIENCES = {
  mobile: () => window.innerWidth < 600,
  desktop: () => window.innerWidth >= 600,
  'new-visitor': () => !localStorage.getItem('returning'),
};

// For projects with the plugin system
window.hlx.plugins.add('experimentation', {
  condition: () =>
    getMetadata('experiment')
    || Object.keys(getAllMetadata('campaign')).length
    || Object.keys(getAllMetadata('audience')).length,
  options: { audiences: AUDIENCES },
  url: '/plugins/experimentation/src/index.js',
});
```

For projects without the plugin system, import manually in `loadEager()`:

```javascript
async function loadEager(doc) {
  if (
    getMetadata('experiment')
    || Object.keys(getAllMetadata('campaign')).length
    || Object.keys(getAllMetadata('audience')).length
  ) {
    const { loadEager: runEager } = await import(
      '../plugins/experimentation/src/index.js'
    );
    await runEager(document, { audiences: AUDIENCES }, pluginContext);
  }
  // ...rest of loadEager
}
```

And in `loadLazy()`:

```javascript
async function loadLazy(doc) {
  const { loadLazy: runLazy } = await import(
    '../plugins/experimentation/src/index.js'
  );
  await runLazy(document, { audiences: AUDIENCES }, pluginContext);
  // ...rest of loadLazy
}
```

### Plugin Configuration Options

```javascript
{
  basePath: '/plugins/experimentation',   // plugin directory
  prodHost: 'www.example.com',            // production hostname
  isProd: () => window.location.hostname === 'www.example.com',
  rumSamplingRate: 10,                    // 10x normal sampling for experiments
  storage: sessionStorage,               // persist variant selection
  audiences: AUDIENCES,                  // audience resolver map
  audiencesMetaTagPrefix: 'audience',    // metadata prefix for audiences
  campaignsMetaTagPrefix: 'campaign',    // metadata prefix for campaigns
  experimentsMetaTag: 'experiment',      // metadata tag name for experiments
  experimentsQueryParameter: 'experiment', // URL query parameter
}
```

### Setting Up A/B Test Experiments

Experiments use a **control** page (the original) and one or more **challenger** pages (variants). The control page carries metadata that defines the experiment.

#### Step 1: Create Challenger Pages

Place challenger pages in `/experiments/<experiment-id>/`:

```
/my-page                            ← control page
/experiments/hero-test/challenger-1  ← first variant
/experiments/hero-test/challenger-2  ← second variant
```

#### Step 2: Add Metadata to the Control Page

Add a metadata block at the bottom of the control page document:

| Metadata          | Value                                              |
|-------------------|----------------------------------------------------|
| Experiment        | hero-test                                          |
| Experiment Variants | /experiments/hero-test/challenger-1, /experiments/hero-test/challenger-2 |

Multiple challenger URLs are separated by commas or line breaks.

#### Step 3: Configure Traffic Split (Optional)

By default, traffic splits evenly across all variants including control. For custom splits, add:

| Metadata           | Value     |
|--------------------|-----------|
| Experiment Split   | 20, 30    |

This allocates 20% to challenger-1, 30% to challenger-2, and the remaining 50% to control. If percentages sum to 100%, control receives 0% traffic.

#### Step 4: Set Experiment Status and Schedule

| Metadata              | Value                        |
|-----------------------|------------------------------|
| Experiment Status     | Active                       |
| Experiment Start Date | 2025-01-15                   |
| Experiment End Date   | 2025-02-15T23:59 GMT+1       |

Status values: `Active`/`True`/`On` or `Inactive`/`False`/`Off`. Flexible date formats supported.

#### Step 5: Preview and Publish

Preview both control and challenger pages via the Sidekick, then publish them all.

### Code-Level Experiments (No Content Variants)

For testing visual/code changes without separate pages, specify a number of variants instead of URLs:

| Metadata             | Value |
|----------------------|-------|
| Experiment           | cta-color-test |
| Experiment Variants  | 2     |

This generates CSS classes on `<body>`:
- `experiment-cta-color-test variant-control`
- `experiment-cta-color-test variant-challenger-1`
- `experiment-cta-color-test variant-challenger-2`

Use these classes in CSS:

```css
/* Control: default blue CTA */
.button.primary {
  background-color: var(--color-blue);
}

/* Challenger 1: green CTA */
body.variant-challenger-1 .button.primary {
  background-color: var(--color-green);
}

/* Challenger 2: orange CTA */
body.variant-challenger-2 .button.primary {
  background-color: var(--color-orange);
}
```

### Audience-Based Content Personalization

Audiences allow serving different content to different visitor segments without running an experiment.

#### Define Audiences in scripts.js

```javascript
const AUDIENCES = {
  mobile: () => window.innerWidth < 600,
  desktop: () => window.innerWidth >= 600,
  'us-visitor': () => navigator.language.startsWith('en-US'),
  'returning': () => localStorage.getItem('visited') === 'true',
};
```

#### Configure Audience Content in Metadata

| Metadata        | Value                              |
|-----------------|------------------------------------|
| Audience        | mobile                             |
| Audience-mobile | /content-variations/mobile-hero    |

When the `mobile` audience resolves, the page content is replaced with the referenced variant.

#### Restrict Experiments to Audiences

| Metadata             | Value        |
|----------------------|--------------|
| Experiment           | mobile-cta   |
| Experiment Variants  | /experiments/mobile-cta/challenger |
| Experiment Audience  | mobile, tablet |

Multiple audiences are treated as OR logic. For AND logic, define a composite audience:

```javascript
const AUDIENCES = {
  'us-mobile': () => window.innerWidth < 600 && navigator.language.startsWith('en-US'),
};
```

### Campaign/URL Parameter Targeting

Campaigns target content based on URL parameters for marketing channels:

| Metadata              | Value                          |
|-----------------------|--------------------------------|
| Campaign-email-promo  | /campaigns/email-promo/landing |

Visitors arriving with `?campaign=email-promo` see the campaign variant. Campaign metadata uses the prefix `campaign-<name>`.

### SEO for Experiment Pages

Prevent experiment variants from being indexed. Use bulk metadata:

| URL                | robots            |
|--------------------|-------------------|
| /experiments/**    | noindex,nofollow  |

Or add `robots: noindex,nofollow` metadata to each challenger page individually.

### RUM (Real User Monitoring) Data Collection

EDS uses Operational Telemetry (formerly RUM) to measure experiment effectiveness. Key checkpoint events:

| Checkpoint    | Description                                    |
|---------------|------------------------------------------------|
| `top`         | Initial JavaScript execution begins            |
| `cwv`         | Core Web Vitals readings collected              |
| `lcp`         | Largest Contentful Paint recorded               |
| `click`       | User interaction with page elements             |
| `formsubmit`  | Form submission with action URL                 |
| `error`       | Unhandled JavaScript errors                     |

Experiment variants automatically get a 10x higher sampling rate than regular pages for statistical significance.

### Conversion Tracking

Default conversion: any click event on the page. Customize with:

| Metadata                     | Value         |
|------------------------------|---------------|
| Experiment Conversion Name   | signup-click  |

For advanced tracking, integrate the `aem-rum-conversion` plugin.

### Integration with Analytics Platforms

#### Adobe Analytics / Adobe Data Layer

```javascript
if (window.hlx.experiment) {
  window.adobeDataLayer = window.adobeDataLayer || [];
  window.adobeDataLayer.push({
    event: 'experiment-applied',
    experiment: {
      id: window.hlx.experiment.id,
      variant: window.hlx.experiment.selectedVariant,
    },
  });
}
```

#### Google Analytics / GTM

```javascript
if (window.hlx.experiment) {
  window.dataLayer = window.dataLayer || [];
  window.dataLayer.push({
    event: 'experiment_view',
    experiment_id: window.hlx.experiment.id,
    experiment_variant: window.hlx.experiment.selectedVariant,
  });
}
```

### Adobe Target Integration

For enterprise personalization, integrate Adobe Target with EDS. Two approaches:

**Adobe Experience Platform WebSDK (recommended):**
- Requires `orgId` and `datastreamId` from Adobe Experience Platform
- Add `alloy.js` to the repository and initialize in `scripts.js`
- Add `Target: on` metadata to pages requiring personalization
- Expected LCP impact: +0.5s desktop (first call), +0.1s subsequent

**Legacy at.js:**
- Requires `clientCode`, `serverDomain`, and `imsOrgId`
- Add optimized `at.js` and initialize in `scripts.js`

Enable Target selectively on pages that need it:

| Metadata | Value |
|----------|-------|
| Target   | on    |

### Experiment-Aware Block Patterns

Blocks can respond to active experiments:

```javascript
export default function decorate(block) {
  // Check if an experiment is active
  const experiment = window.hlx?.experiment;
  if (experiment) {
    block.dataset.experiment = experiment.id;
    block.dataset.variant = experiment.selectedVariant;
  }

  // Variant-specific logic
  const variant = document.body.classList.contains('variant-challenger-1');
  if (variant) {
    // Render alternative layout
    block.classList.add('compact-layout');
  }
}
```

### Accessing Experiment State

```javascript
// Current experiment details
const experiment = window.hlx.experiment;
// { id, selectedVariant, variants, ... }

// Current audience resolution
const audience = window.hlx.audience;

// Current campaign
const campaign = window.hlx.campaign;
```

### Performance Impact of Experiments

- The experimentation plugin loads only when experiment/audience/campaign metadata is detected
- Variant resolution happens in `loadEager()` before blocks render — no content flicker
- RUM data collection is non-blocking and deferred
- Adobe Target integration adds LCP latency: ~0.5-1.3s on first load
- Recommendation: only enable Target on pages that require it

### Anti-Patterns

- **Too many variants** — Each additional variant dilutes traffic; keep to 2-3 variants maximum for statistical significance in reasonable timeframes
- **Experiments on critical path** — Never block page rendering waiting for experiment resolution; the plugin handles this correctly when loaded in `loadEager()`
- **Not setting experiment end dates** — Always define `Experiment End Date` to avoid running experiments indefinitely
- **Testing too many things at once** — Each variant should change one element; multi-variable changes make it impossible to attribute results
- **Forgetting to noindex challengers** — Experiment pages in `/experiments/` must have `robots: noindex,nofollow` to prevent SEO issues
- **Client-side audience resolution with PII** — Audience functions run in the browser; never fetch or expose personal data
- **Ignoring sample size** — Check that your traffic volume supports the number of variants before launching
- **Not publishing all pages** — Both control and all challenger pages must be published for the experiment to work
