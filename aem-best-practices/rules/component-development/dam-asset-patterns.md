---
title: AEM Assets and DAM Patterns
impact: HIGH
impactDescription: Asset microservices replace custom DAM workflows on Cloud Service — incorrect patterns cause missing renditions, slow delivery, and wasted compute
tags: dam, assets, microservices, processing-profiles, metadata, renditions, smart-tags, asset-compute, connected-assets
---

## AEM Assets and DAM Patterns

AEM as a Cloud Service processes assets via Asset Microservices (not DAM Update Asset workflow). Custom renditions require Asset Compute workers. Understanding the cloud-native asset pipeline is essential for proper DAM implementation.

---

### 1. Asset Microservices Architecture

```
Upload → Asset Microservices (Adobe I/O) → Renditions stored in AEM
         ├── Standard renditions (thumbnail, web, etc.)
         ├── Processing Profile renditions (custom sizes, crops)
         ├── Smart Tags (AI-based tagging)
         ├── Text extraction (PDF, Office docs)
         └── Video transcoding (Dynamic Media)

Key difference from 6.5:
  AEM 6.5:  DAM Update Asset workflow → custom steps → renditions
  Cloud:    Asset Microservices → Processing Profiles → renditions
            (no custom Java workflow steps for asset processing)
```

---

### 2. Processing Profiles

#### Image Processing Profile

Configure via: **Tools → Assets → Processing Profiles**

```
Profile: "mysite-images"
Renditions:
  ├── web-optimized  → Width: 1280, Format: WebP, Quality: 80
  ├── thumbnail-lg   → Width: 600, Height: 400, Crop: Smart Crop
  ├── thumbnail-sm   → Width: 300, Height: 200, Crop: Smart Crop
  └── social-share   → Width: 1200, Height: 630, Crop: Center
```

**Apply profile to folder:**

```
/content/dam/mysite
  └── jcr:content
        └── processing/profile → /conf/global/settings/dam/processing/profile/mysite-images
```

#### Video Processing Profile

```
Profile: "mysite-video"
Encoding Presets:
  ├── MP4 720p  → H.264, 720p, 2.5 Mbps
  ├── MP4 1080p → H.264, 1080p, 5 Mbps
  └── HLS       → Adaptive bitrate streaming
```

---

### 3. Custom Asset Compute Workers

For custom rendition logic beyond processing profiles, use Asset Compute SDK (Adobe App Builder):

#### Worker Implementation

```javascript
// worker.js — Asset Compute worker
'use strict';

const { worker, SourceCorruptError } = require('@adobe/asset-compute-sdk');
const { serializeXmp } = require('@adobe/asset-compute-xmp');
const sharp = require('sharp');

exports.main = worker(async (source, rendition, params) => {
  // Read source asset
  const input = await source.read();

  // Process with sharp (or any Node.js library)
  const output = await sharp(input)
    .resize(rendition.instructions.width || 800)
    .webp({ quality: rendition.instructions.quality || 80 })
    .toBuffer();

  // Write rendition
  await rendition.write(output);
}, {
  // Supported MIME types
  supportedInputFormats: ['image/jpeg', 'image/png', 'image/tiff'],
});
```

#### Worker Manifest

```yaml
# manifest.yml
packages:
  mysite-asset-compute:
    actions:
      watermark-worker:
        function: actions/watermark/worker.js
        runtime: nodejs:18
        limits:
          memorySize: 256
          timeout: 300000
        inputs:
          watermarkUrl: "https://example.com/watermark.png"
```

#### Register Worker as Processing Profile

```
Tools → Assets → Processing Profiles → Custom tab
  Worker URL: https://12345.adobeioruntime.net/api/v1/web/mysite-asset-compute/watermark-worker
  Rendition Name: watermarked
  Extension: png
```

---

### 4. Metadata Schemas

#### Custom Metadata Schema

Configure via: **Tools → Assets → Metadata Schemas**

```xml
<!-- Custom metadata tab for product assets -->
<!-- /conf/mysite/settings/dam/adminui-extension/metadataschema/mysite-product -->
<jcr:root xmlns:jcr="http://www.jcp.org/jcr/1.0" xmlns:sling="http://sling.apache.org/jcr/sling/1.0">
    <items jcr:primaryType="nt:unstructured">
        <product-info
            jcr:primaryType="nt:unstructured"
            jcr:title="Product Info"
            sling:resourceType="dam/gui/coral/components/admin/schemaforms/formtab">
            <items jcr:primaryType="nt:unstructured">
                <sku
                    jcr:primaryType="nt:unstructured"
                    sling:resourceType="dam/gui/coral/components/admin/schemaforms/formbuilder/textfield"
                    fieldLabel="Product SKU"
                    name="./jcr:content/metadata/product-sku"
                    emptyText="Enter SKU"/>
                <category
                    jcr:primaryType="nt:unstructured"
                    sling:resourceType="dam/gui/coral/components/admin/schemaforms/formbuilder/dropdown"
                    fieldLabel="Category"
                    name="./jcr:content/metadata/product-category">
                    <options jcr:primaryType="nt:unstructured">
                        <option1 jcr:primaryType="nt:unstructured" text="Electronics" value="electronics"/>
                        <option2 jcr:primaryType="nt:unstructured" text="Clothing" value="clothing"/>
                    </options>
                </category>
            </items>
        </product-info>
    </items>
</jcr:root>
```

#### Folder Metadata Schema

Apply metadata rules at the folder level:

```
/content/dam/mysite/products
  └── jcr:content
        └── metadata
              └── metadataSchema → /conf/mysite/settings/dam/adminui-extension/metadataschema/mysite-product
```

---

### 5. Programmatic Asset Access

#### Reading Asset Data (Sling Model)

```java
@Model(adaptables = SlingHttpServletRequest.class, adapters = AssetInfo.class,
       resourceType = "myproject/components/asset-card")
public class AssetInfoImpl implements AssetInfo {

    @ValueMapValue
    @Via("resource")
    private String fileReference;

    @SlingObject
    private ResourceResolver resourceResolver;

    @Override
    public String getTitle() {
        Asset asset = getAsset();
        return asset != null ? asset.getMetadataValue(DamConstants.DC_TITLE) : "";
    }

    @Override
    public String getAltText() {
        Asset asset = getAsset();
        if (asset == null) return "";
        // Check dc:description first, then dc:title
        String alt = asset.getMetadataValue(DamConstants.DC_DESCRIPTION);
        return StringUtils.isNotBlank(alt) ? alt : asset.getMetadataValue(DamConstants.DC_TITLE);
    }

    @Override
    public Map<String, String> getRenditions() {
        Asset asset = getAsset();
        if (asset == null) return Collections.emptyMap();

        Map<String, String> renditions = new LinkedHashMap<>();
        for (Rendition rendition : asset.getRenditions()) {
            renditions.put(rendition.getName(), rendition.getPath());
        }
        return renditions;
    }

    private Asset getAsset() {
        if (StringUtils.isBlank(fileReference)) return null;
        Resource assetResource = resourceResolver.getResource(fileReference);
        return assetResource != null ? assetResource.adaptTo(Asset.class) : null;
    }
}
```

#### Web-Optimized Image Delivery

```java
// Use AssetDelivery API for web-optimized URLs (Cloud Service)
@Reference
private AssetDelivery assetDelivery;

public String getOptimizedUrl(Resource assetResource, int width) {
    if (assetDelivery != null) {
        Map<String, Object> params = new HashMap<>();
        params.put("preferwebp", true);
        params.put("width", width);
        params.put("quality", 80);
        return assetDelivery.getDeliveryURL(assetResource, params);
    }
    // Fallback for local SDK
    return assetResource.getPath();
}
```

HTL usage with web-optimized delivery:

```html
<!-- Core Image component handles this automatically -->
<sly data-sly-use.image="com.adobe.cq.wcm.core.components.models.Image">
    <img src="${image.src}" srcset="${image.srcset}" alt="${image.alt}"
         width="${image.width}" height="${image.height}" loading="lazy"/>
</sly>
```

---

### 6. Smart Tags and AI Features

#### Smart Tags Configuration

```
Smart Tags are automatically applied by asset microservices:
├── Visual tags — objects, scenes, colors detected in images
├── Text tags — extracted from documents (PDF, DOCX)
└── Custom training — upload training assets for domain-specific tags

Access: Asset properties → Basic tab → Tags section
API: GET /api/assets/{path}.json → metadata.predictedTags
```

#### Accessing Smart Tags Programmatically

```java
public List<String> getSmartTags(Resource assetResource) {
    Asset asset = assetResource.adaptTo(Asset.class);
    if (asset == null) return Collections.emptyList();

    Resource metadataResource = assetResource.getChild("jcr:content/metadata/predictedTags");
    if (metadataResource == null) return Collections.emptyList();

    List<String> tags = new ArrayList<>();
    for (Resource tagResource : metadataResource.getChildren()) {
        ValueMap tagProps = tagResource.getValueMap();
        String name = tagProps.get("name", String.class);
        double confidence = tagProps.get("confidence", 0.0);
        if (confidence > 0.7) { // Only high-confidence tags
            tags.add(name);
        }
    }
    return tags;
}
```

---

### 7. Asset Selector (UI Integration)

For picking assets in custom UIs:

```html
<!-- Micro-Frontend Asset Selector -->
<script src="https://experience.adobe.com/solutions/CQ-assets-selectors/assets/resources/assets-selectors.js"></script>

<script>
const assetSelector = new AssetSelector({
    imsOrg: 'YOUR_IMS_ORG@AdobeOrg',
    repositoryId: 'author-p12345-e67890.adobeaemcloud.com',
    onAssetSelected: (asset) => {
        console.log('Selected:', asset.path, asset.name);
        // Use asset.url for delivery URL
    },
    filterSchema: [
        { header: 'File Type', groupKey: 'TopGroup', fields: [
            { element: 'checkbox', name: 'type', options: [
                { label: 'Images', value: 'image/*' },
                { label: 'Videos', value: 'video/*' }
            ]}
        ]}
    ]
});
assetSelector.open();
</script>
```

---

### 8. Connected Assets

Share assets across Sites and Assets programs:

```
Sites instance (author) ←── Connected Assets ──→ Assets instance (DAM)

Configuration:
  Tools → Cloud Services → Connected Assets
  Remote DAM URL: https://author-assets-p12345-e67890.adobeaemcloud.com

Access:
  Page Editor → Asset Finder → Shows remote assets
  Authors can drag-drop remote assets into pages
```

---

### 9. Anti-Patterns

#### Custom DAM Workflows on Cloud Service

```
// WRONG — customizing DAM Update Asset workflow (has no effect on Cloud)
/var/workflow/models/dam/update_asset → add custom step

// CORRECT — use Processing Profiles for standard renditions
// CORRECT — use Asset Compute workers for custom processing
```

#### Storing Large Binaries Directly

```java
// WRONG — storing base64 or binary in JCR properties
node.setProperty("thumbnail", base64String); // bloats JCR

// CORRECT — store in DAM and reference
// /content/dam/mysite/generated/thumbnail.jpg
// Reference via fileReference property
```

#### Ignoring Web-Optimized Delivery

```html
<!-- WRONG — direct DAM path (serves original, no optimization) -->
<img src="/content/dam/mysite/hero.jpg">

<!-- CORRECT — use Core Image component or AssetDelivery API -->
<!-- Gets WebP, proper sizing, CDN delivery -->
<sly data-sly-use.image="com.adobe.cq.wcm.core.components.models.Image">
    <img src="${image.src}" srcset="${image.srcset}" loading="lazy"/>
</sly>
```

#### Processing Assets Synchronously

```java
// WRONG — processing assets in a servlet or workflow step
BufferedImage img = ImageIO.read(asset.getOriginal().getStream());
// Resize, watermark, etc. — consumes AEM heap memory

// CORRECT — use Asset Compute workers (runs in Adobe I/O Runtime)
// Or use Dynamic Media image modifiers for on-the-fly transforms
```
