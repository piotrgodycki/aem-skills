---
title: Dynamic Media and Asset Delivery for AEM Cloud Service
impact: CRITICAL
impactDescription: Unoptimized image and media delivery is the single largest contributor to poor Core Web Vitals (LCP, CLS), excessive bandwidth consumption, and degraded user experience on AEM sites
tags: dynamic-media, scene7, smart-imaging, image-presets, responsive-images, webp, avif, video-streaming, image-optimization, web-optimized-image-delivery, core-image-component, srcset, lazy-loading, fetchpriority, cdn, dam, 3d-assets, viewer-presets, adaptive-image-servlet, performance, core-web-vitals
---

## Dynamic Media and Asset Delivery

Comprehensive guide to delivering optimized images, video, and rich media from AEM Cloud Service. Covers both Dynamic Media (Scene7) and the newer Web-Optimized Image Delivery (WOID) approach, with practical patterns for responsive images, CDN caching, and performance tuning.

---

### 1. Dynamic Media Overview (Scene7 Mode)

AEM Cloud Service uses Dynamic Media in Scene7 mode exclusively. When an asset is uploaded to AEM, it is replicated to Dynamic Media for processing and rendition generation. All size variations and visual effects are created dynamically at delivery time from a single primary source file.

**Key architecture points:**

- AEM is the primary source asset repository; Dynamic Media handles processing and delivery
- Assets are served from the Scene7 CDN (`https://<company>.scene7.com/is/image/...`)
- One master asset produces unlimited dynamic renditions via URL parameters
- Smart Imaging auto-negotiates format (AVIF/WebP) and applies DPR optimization
- Dynamic Media is NOT HIPAA-ready and cannot be used when Enhanced Security is enabled

**When to use Dynamic Media vs WOID:**

| Capability | Dynamic Media (Scene7) | Web-Optimized Image Delivery |
|---|---|---|
| Smart Imaging (auto AVIF/WebP) | Yes | WebP only |
| Image presets / URL modifiers | Yes (100+ commands) | No |
| Smart Crop (AI) | Yes | No |
| Video adaptive streaming | Yes | No |
| Spin sets, mixed media, 3D | Yes | No |
| Requires DM license | Yes | No (included in AEMaaCS) |
| Works with Core Components | Yes | Yes |
| Headless / Content Fragment delivery | Via URL | Via delivery API |

---

### 2. Image Profiles and Processing

Image Profiles define how assets are processed when uploaded to specific DAM folders. They control smart crop, unsharp mask sharpening, and rendition generation.

**Smart Crop configuration best practices:**

- Create 10-15 smart crops per profile for balanced screen coverage (maximum supported: 100)
- Name crops by dimensions (e.g., `1200x600`) rather than usage context to minimize duplicates
- Minimum source image size: 50 x 50 pixels
- Apply profiles to specific folders rather than the DAM root when possible
- Supported formats: BMP, JPEG, PNG, PSD, TIFF (not GIF animations, PDF, or INDD)

**Unsharp Mask settings for web delivery:**

```
Amount: 1.75 (range 0-5)
Radius: 0.3  (range 0-250 pixels; 0.2-0.3 for web)
Threshold: 2  (range 0-255; prevents noise on smooth areas)
```

The unsharp mask is applied only to downscaled PTIFF renditions that are downsampled more than 50%.

**Correct -- apply profile to specific product image folders:**

Navigate to Tools > Assets > Image Profiles, create a profile with smart crop breakpoints for your design system (e.g., 320, 640, 960, 1280, 1920), then apply it to `/content/dam/mysite/products/`.

**Incorrect -- applying a single global profile to the DAM root:**

This processes every asset (PDFs, documents, icons) with unnecessary smart crops, wasting processing resources.

---

### 3. Image Serving URL Syntax and Modifiers

Dynamic Media URLs follow the Image Serving API protocol. The base URL structure:

```
https://<company>.scene7.com/is/image/<company>/<asset-path>?<modifiers>
```

**Essential URL modifiers:**

| Modifier | Syntax | Purpose |
|---|---|---|
| `wid` | `wid=800` | Output width in pixels |
| `hei` | `hei=600` | Output height in pixels |
| `fmt` | `fmt=webp-alpha` | Output format (jpeg, png, png-alpha, webp, webp-alpha, avif, gif, tif, pdf) |
| `qlt` | `qlt=85,0` | JPEG/WebP quality (1-100). Second param: 0=no chroma subsampling, 1=subsampling |
| `crop` | `crop=100,50,400,300` | Crop region: x,y,width,height |
| `op_sharpen` | `op_sharpen=1` | Simple sharpening |
| `op_usm` | `op_usm=1.75,0.3,2,0` | Unsharp mask: amount,radius,threshold,monochrome |
| `resMode` | `resMode=sharp2` | Resampling algorithm (sharp2 recommended for web) |
| `bgc` | `bgc=ffffff` | Background color (hex) |
| `scl` | `scl=2` | Scale factor |
| `flip` | `flip=lr` | Flip: lr (left-right), ud (up-down) |
| `rotate` | `rotate=90` | Rotation in degrees |
| `size` | `size=400,300` | Width,height reference |

**Format syntax detail:**

```
fmt=format[,pixelType][,compression]

# pixelType: rgb (default), gray, cmyk
# compression: jpeg, lossy, lossless, lzw, zip, none

# Examples:
fmt=webp            # WebP lossy (default)
fmt=webp,,lossless  # WebP lossless
fmt=png-alpha       # PNG with alpha channel
fmt=avif            # AVIF format
fmt=jpeg            # JPEG (default when no fmt specified)
```

**Correct -- optimized image URL with recommended baseline:**

```
https://myco.scene7.com/is/image/myco/hero-banner?wid=1200&hei=600&fmt=jpg&qlt=85,0&resMode=sharp2&op_usm=1.75,0.3,2,0
```

**Correct -- using an image preset instead of inline modifiers:**

```
https://myco.scene7.com/is/image/myco/hero-banner?$preset_hero$
```

**Incorrect -- no size constraints, no quality settings, original format:**

```
https://myco.scene7.com/is/image/myco/hero-banner
```

This serves the full-resolution original, potentially delivering a 5000x3000 px image to a 375px-wide phone.

**Note:** `icc=` and `iccEmbed=` color correction commands and basic templating/text rendering commands are not supported in Dynamic Media on AEM Cloud Service.

---

### 4. Smart Imaging

Smart Imaging automatically optimizes image delivery by combining Adobe Sensei AI with the CDN. It is applied at delivery time, not at processing time.

**What Smart Imaging does:**

- **Auto-format negotiation:** Detects browser capabilities and serves AVIF > WebP > JPEG/PNG in order of preference
- **Device Pixel Ratio (DPR) optimization:** Adjusts image resolution for high-DPI screens
- **Network bandwidth optimization:** Reduces quality dynamically on slow connections
- Requires no code changes -- it operates on the CDN layer

**Auto-format priority (when enabled):**

1. AVIF -- if browser supports it AND license allows it
2. WebP -- if AVIF is not supported or not licensed
3. JPEG -- if neither modern format is supported and image has no alpha
4. PNG -- if image has alpha channel and browser lacks modern format support

**Correct -- let Smart Imaging choose the format:**

```
https://myco.scene7.com/is/image/myco/product-photo?wid=800&qlt=85
```

Smart Imaging will negotiate AVIF or WebP automatically based on the client browser Accept header.

**Incorrect -- overriding the format when Smart Imaging is active:**

```
https://myco.scene7.com/is/image/myco/product-photo?wid=800&qlt=85&fmt=webp
```

Hardcoding `fmt=webp` bypasses Smart Imaging. You lose AVIF delivery for browsers that support it and force WebP on browsers that do not.

**DPR behavior:**

Smart Imaging automatically applies DPR scaling. For a `wid=400` request on a 2x DPR device, it serves an 800px image. This can be controlled with:

```
https://myco.scene7.com/is/image/myco/photo?wid=400&dpr=off
```

---

### 5. Responsive Image Delivery

#### Using Dynamic Media with breakpoints

When adding Dynamic Media assets to pages via the Dynamic Media component, configure breakpoints as a comma-separated list. The component generates responsive `<img>` elements with the correct `srcset`.

**Correct -- responsive image with DM breakpoints:**

```html
<!-- Configure breakpoints in the Dynamic Media component dialog: 320,640,960,1280,1920 -->
<!-- Resulting output: -->
<img src="https://myco.scene7.com/is/image/myco/hero?wid=1920&hei=800&qlt=85"
     srcset="https://myco.scene7.com/is/image/myco/hero?wid=320&hei=133&qlt=85 320w,
             https://myco.scene7.com/is/image/myco/hero?wid=640&hei=267&qlt=85 640w,
             https://myco.scene7.com/is/image/myco/hero?wid=960&hei=400&qlt=85 960w,
             https://myco.scene7.com/is/image/myco/hero?wid=1280&hei=533&qlt=85 1280w,
             https://myco.scene7.com/is/image/myco/hero?wid=1920&hei=800&qlt=85 1920w"
     sizes="100vw"
     alt="Descriptive alt text for hero image"
     loading="lazy" />
```

#### Using the `<picture>` element for art direction

When different crops are needed at different breakpoints (not just different sizes), use the `<picture>` element with Smart Crop presets:

```html
<picture>
    <!-- Mobile: square crop -->
    <source media="(max-width: 767px)"
            srcset="https://myco.scene7.com/is/image/myco/hero%3Asquare?wid=750&qlt=85" />
    <!-- Tablet: 4:3 crop -->
    <source media="(max-width: 1023px)"
            srcset="https://myco.scene7.com/is/image/myco/hero%3Atablet?wid=1024&qlt=85" />
    <!-- Desktop: wide banner crop -->
    <img src="https://myco.scene7.com/is/image/myco/hero%3Abanner?wid=1920&qlt=85"
         alt="Hero banner with product showcase" />
</picture>
```

The `%3A` encodes the colon separator between the image name and the Smart Crop rendition name.

#### Dynamic Media Responsive Image Library

For client-side responsive control, use the DM Responsive Image Library with `data-breakpoints`:

```html
<img src="https://myco.scene7.com/is/image/myco/product"
     data-src="https://myco.scene7.com/is/image/myco/product"
     data-breakpoints="360:wid=360&qlt=80,768:wid=768&qlt=85,1200:wid=1200&qlt=85"
     alt="Product image" />

<script src="https://myco.scene7.com/s7viewers/libs/responsive_image.js"></script>
```

**Incorrect -- fixed-width image for all devices:**

```html
<img src="https://myco.scene7.com/is/image/myco/hero?wid=1920&hei=800"
     alt="Hero image" />
```

This forces a 1920px image on all devices, wasting bandwidth on mobile.

---

### 6. Image Presets vs URL Modifiers

**Image Presets** are reusable, named collections of image transformations managed in AEM (Tools > Assets > Image Presets). They appear in the URL as `$preset_name$`.

**When to use Image Presets:**

- Consistent sizing across the site (thumbnails, cards, heroes)
- Non-technical authors need to select rendering options
- Centralized control -- changing a preset updates all images using it
- Fewer URL parameters to manage

**When to use inline URL modifiers:**

- One-off adjustments or overrides
- Dynamic values computed by frontend code (e.g., exact container width)
- Combining a preset with additional modifiers

**Correct -- preset for consistent card images across the site:**

```
https://myco.scene7.com/is/image/myco/product-001?$card_thumbnail$
```

Where `$card_thumbnail$` is defined as: `wid=400&hei=400&fit=crop&qlt=85&resMode=sharp2`

**Correct -- preset with additional modifier override:**

```
https://myco.scene7.com/is/image/myco/product-001?$card_thumbnail$&qlt=90
```

The inline `qlt=90` overrides the preset's `qlt=85` value.

**Image presets are automatically published** and immediately available as dynamic renditions. No manual publish step is needed.

---

### 7. Video Delivery

Dynamic Media provides adaptive video streaming with automatic quality adjustment.

**Video specifications:**

- Maximum length: 30 minutes
- Maximum input resolution: 16,384 x 16,384
- Maximum output resolution: 8,192 x 4,320
- Maximum file size: 15 GB
- Output codec: MP4 H.264
- Streaming protocols: HLS, DASH, progressive HTTP/HTTPS

**Adaptive Video Encoding:**

Apply the built-in Adaptive Video Encoding profile, which encodes at multiple bitrates (400, 800, 1000+ kbps). The video viewer automatically selects quality based on available bandwidth.

**Correct -- embed adaptive video with the Dynamic Media component:**

```html
<!-- In AEM page: use the Dynamic Media component with a video viewer preset -->
<!-- The component outputs an embed similar to: -->
<div class="s7video"
     data-asset="myco/product-demo"
     data-viewer="VideoViewer"
     style="width:100%;height:auto;">
</div>

<!-- Or use the embed URL directly: -->
<iframe src="https://myco.scene7.com/s7viewers/html5/VideoViewer.html?asset=myco/product-demo&serverUrl=https://myco.scene7.com/is/image/&contenturl=https://myco.scene7.com/is/content/&config=myco/Universal_HTML5_Video&autoplay=0"
        frameborder="0"
        allowfullscreen
        style="width:100%;aspect-ratio:16/9;">
</iframe>
```

**Video best practices:**

- Use the predefined Adaptive Video Encoding profile unless you have specific bitrate requirements
- Set `autoplay=0` or use muted autoplay (`autoplay=1&muted=1`) -- browsers block unmuted autoplay
- Add WebVTT captions (`.vtt` files) for accessibility; Dynamic Media supports AI-generated captions in 60+ languages
- Multiple audio tracks are supported (`.mp3` format)
- Videos stream over HTTPS by default

**Incorrect -- uploading pre-encoded single-bitrate MP4s:**

This defeats adaptive streaming. Upload the highest quality source and let Dynamic Media encode the adaptive set.

---

### 8. Dynamic Media Viewers

Available viewer types for different media experiences:

| Viewer | Use Case |
|---|---|
| Image Viewer | Single image with pan/zoom |
| Zoom Viewer | Product detail with flyout or inline zoom |
| Spin Set Viewer | 360-degree product rotation |
| Mixed Media Viewer | Combines images, spin sets, and video in one viewer |
| Carousel Viewer | Shoppable banners with hotspots (non-zoomable) |
| Video Viewer | Adaptive streaming video with controls |
| Panoramic Image | 360-degree spherical panoramic images |
| Interactive Image | Images with clickable hotspots |
| Interactive Video | Video with shoppable overlays |
| Video 360 / VR | Equirectangular 360/VR video |
| 3D / Dimensional | Interactive 3D model (GLB, OBJ, STL) |

**Custom Viewer Presets:**

Create custom viewer presets at Tools > Assets > Viewer Presets > Create. Each viewer type supports CSS customization and configuration attributes.

**Correct -- ensure viewer assets are synced before use:**

All viewer assets (icons, CSS, presets) must be synced with Dynamic Media before using players. Only include the primary viewer JavaScript file on pages -- do not reference additional JS files that the viewer runtime may download (such as the HTML5 SDK `Utils.js` library).

---

### 9. 360/3D Asset Viewer Patterns

**Supported 3D formats:**

| Format | Extension | MIME Type | Notes |
|---|---|---|---|
| GLB | .glb | model/gltf-binary | Includes materials and textures in single file (preferred) |
| OBJ | .obj | application/x-tgif | Standard 3D object format |
| STL | .stl | application/vnd.ms-pki.stl | Stereolithography / 3D printing |
| USDZ | .usdz | model/vnd.usdz+zip | Ingestion only -- no viewer/interaction support |

**Correct -- add 3D asset to page using the 3D Media component:**

1. Enable the 3D Media component in the page template's policy (Layout Container > Allowed Components > Dynamic Media)
2. Drag the 3D Media component onto the page
3. Select the GLB asset from the asset panel
4. The Dimensional viewer preset is applied automatically

```html
<!-- Rendered output uses the Dimensional viewer: -->
<div class="s7dimensionalviewer"
     data-asset="myco/product-model"
     style="width:100%;aspect-ratio:1/1;">
</div>
```

**360 Video:**

- Supported extensions: .mp4, .mkv, .mov (H.264 codec)
- No additional configuration required beyond using the Video 360 Media component
- Use the Video360Viewer preset

**Browser compatibility note:** The 3D Media component and 3D asset preview may have compatibility issues with some Chrome versions. Test with Firefox and Safari as alternatives.

---

### 10. Web-Optimized Image Delivery (WOID)

WOID is the AEM-native image optimization approach that does NOT require a Dynamic Media license. It is included in AEM as a Cloud Service and delivers DAM images in WebP format with approximately 25% smaller file sizes.

**How it works:**

- Server-driven content negotiation selects WebP for supporting browsers, falls back to JPEG/PNG
- Image URLs keep their original extensions (.jpg/.png) -- WebP is served via content negotiation, not URL changes
- No `<picture>` or `srcset` elements are generated specifically for format switching
- Maximum supported: images up to 50 MB and 12,000 x 12,000 pixels
- The service selects the largest rendition <= 2048px as the processing base, then applies requested width without upscaling

**Enabling WOID:**

1. Edit the page template
2. Open the Image Component design dialog
3. Check "Enable Web Optimized Images"
4. Available for Image Component v1, v2, and v3

**Limitations:**

- Only processes images stored under `/content/dam` -- images stored directly in `cq:Page` nodes fall back to the Adaptive Image Servlet
- Only available in AEM as a Cloud Service (not on-premise, not local SDK)
- Dispatcher must not block requests to `/adobe/*` paths

**Custom components can use the WOID API:**

```java
import com.adobe.cq.wcm.spi.AssetDelivery;

@Reference
private AssetDelivery assetDelivery;

// Get optimized delivery URL
String deliveryUrl = assetDelivery.getDeliveryURL(
    resource,                    // DAM asset resource
    Map.of("width", "800",       // Desired width
            "preferwebp", "true") // Request WebP format
);
```

Do NOT embed the delivery URL directly by constructing it manually -- this bypasses the SPI interface and violates Media Library terms of usage.

---

### 11. Core Image Component

The AEM Core Components Image Component (v3) is the standard for image delivery in AEM Sites. It supports both WOID and Dynamic Media backends.

**Key capabilities:**

- Browser-native responsive images via `srcset` with configurable width breakpoints
- Lazy loading enabled by default (defers image loading until visible in viewport)
- Dynamic Media integration (image presets, smart crops, image modifiers)
- WOID integration (WebP delivery)
- SVG passthrough (served without transformation)
- Schema.org microdata and Adobe Client Data Layer integration

**Correct -- configure width breakpoints in the design dialog:**

In the page template, open the Image Component design dialog and define widths that match your design system breakpoints: e.g., `320, 640, 960, 1280, 1920`.

The component generates:

```html
<div class="cmp-image" itemscope itemtype="http://schema.org/ImageObject">
    <img src="/content/dam/mysite/hero.coreimg.85.1920.jpeg/1680000000000/hero.jpeg"
         srcset="/content/dam/mysite/hero.coreimg.85.320.jpeg/1680000000000/hero.jpeg 320w,
                 /content/dam/mysite/hero.coreimg.85.640.jpeg/1680000000000/hero.jpeg 640w,
                 /content/dam/mysite/hero.coreimg.85.960.jpeg/1680000000000/hero.jpeg 960w,
                 /content/dam/mysite/hero.coreimg.85.1280.jpeg/1680000000000/hero.jpeg 1280w,
                 /content/dam/mysite/hero.coreimg.85.1920.jpeg/1680000000000/hero.jpeg 1920w"
         sizes="100vw"
         loading="lazy"
         alt="Descriptive alt text from DAM metadata"
         itemprop="contentUrl"
         width="1920"
         height="800" />
</div>
```

**Adaptive Image Servlet behavior:**

When WOID is not enabled, the Adaptive Image Servlet handles image delivery:

1. Filters renditions by MIME type (matching the original asset format)
2. Selects the smallest rendition that is >= the requested container width
3. Falls back to the original if no suitable rendition exists
4. Supports `Last-Modified` conditional requests (requires Dispatcher configuration for header caching)

**Correct -- configure lazy loading and fetchpriority for LCP images:**

```html
<!-- Hero/LCP image: disable lazy loading, set high fetchpriority -->
<div class="cmp-image"
     data-cmp-lazy-threshold="0">
    <img src="/content/dam/mysite/hero.coreimg.85.1920.jpeg"
         loading="eager"
         fetchpriority="high"
         alt="Main hero image" />
</div>

<!-- Below-fold images: lazy loading is default -->
<div class="cmp-image">
    <img src="/content/dam/mysite/content-photo.coreimg.85.960.jpeg"
         loading="lazy"
         alt="Supporting content image" />
</div>
```

**Incorrect -- all images loaded eagerly with no size hints:**

```html
<img src="/content/dam/mysite/photo.jpeg"
     alt="Photo" />
<!-- Missing: loading strategy, width/height (causes CLS), srcset, sizes -->
```

---

### 12. DAM Asset Renditions

AEM generates and manages several categories of renditions:

- **Original:** The uploaded source file (never serve directly to end users for large images)
- **Web renditions:** Auto-generated web-optimized renditions at various widths (used by WOID)
- **Dynamic Media renditions:** Generated on-demand via URL parameters (not stored in DAM)
- **Smart Crop renditions:** AI-generated crops defined by Image Profiles
- **Custom renditions:** Created by processing profiles or workflow steps
- **Thumbnail:** Standard `cq5dam.thumbnail.*` renditions for AEM UI

**Best practice:** Store high-resolution originals in DAM (2000+ px on longest side for zoomable content). Let Dynamic Media or WOID generate all delivery renditions dynamically. Never pre-create fixed-size renditions for delivery.

---

### 13. Asset Metadata for Frontend

DAM metadata is critical for accessibility, SEO, and content management.

**Key metadata fields for frontend delivery:**

| Metadata Field | JCR Property | Frontend Usage |
|---|---|---|
| Title | `dc:title` | Image caption, tooltip |
| Description | `dc:description` | Extended alt text, figure caption |
| Alt Text | `dam:scene7altText` or custom | `alt` attribute on `<img>` |
| Tags | `cq:tags` | Filtering, categorization |
| Smart Tags | Auto-generated by AI | Search, content recommendations |

**Correct -- configure Image Component to inherit alt text from DAM:**

The Core Image Component automatically inherits alt text from the DAM asset metadata. Authors can override it in the component dialog. Ensure DAM authors always populate alt text during asset upload workflows.

**Smart Tags:**

- Automatically generated by Adobe Sensei on upload
- Applied to images, video, and text-based assets
- Used for enhanced search and discovery
- Custom smart tags can be trained for domain-specific vocabulary

---

### 14. Content Delivery Network (CDN)

AEM Cloud Service uses multiple CDN layers depending on the delivery path:

**AEM-managed CDN (Fastly):**

- Serves AEM pages, Core Component images (via Adaptive Image Servlet / WOID)
- Configured via `cdn.yaml` in the AEM project
- Cache controlled by `Cache-Control`, `Surrogate-Control`, and `Expires` headers
- Use `s-maxage` in `Surrogate-Control` to set CDN TTL independently of browser cache

**Scene7 CDN (Dynamic Media):**

- Serves all Dynamic Media image and video URLs (`*.scene7.com`)
- Default TTL: 10 hours (configurable in Tools > Assets > Dynamic Media Publish Setup)
- Supports CDN cache invalidation (Tools > Assets > CDN Invalidation)
- Maximum 1000 URLs per invalidation request
- With Smart Imaging enabled, invalidating a base URL also purges all variant URLs with different query parameters

**Correct -- preconnect to Dynamic Media CDN:**

```html
<!-- customheaderlibs.html -->
<link rel="preconnect" href="https://myco.scene7.com" crossorigin />
<link rel="dns-prefetch" href="https://myco.scene7.com" />

<!-- For DM with OpenAPI: -->
<link rel="preconnect" href="https://delivery-pXXXX-eYYYY.adobeaemcloud.com" crossorigin />
```

**Cache-Control for Dispatcher-served images:**

```apache
# dispatcher/src/conf.d/available_vhosts/mysite.vhost
<LocationMatch "^/content/dam/.*\.(jpg|jpeg|png|gif|webp|svg)$">
    Header set Cache-Control "public, max-age=86400, s-maxage=604800"
    Header set Surrogate-Control "max-age=604800"
</LocationMatch>
```

---

### 15. Dynamic Media with OpenAPI Capabilities

The newer Dynamic Media with OpenAPI capabilities provides a REST-based delivery API with a different URL pattern:

**Delivery URL format:**

```
https://delivery-pXXXX-eYYYY.adobeaemcloud.com/adobe/assets/{assetId}/as/{seoName}.{format}?width=800&quality=85
```

**Supported query parameters:**

| Parameter | Description |
|---|---|
| `width` | Output width in pixels |
| `height` | Output height in pixels |
| `quality` | Compression quality (default: 65) |
| `format` | Output format (JPEG, WebP, etc.) |
| `crop` | Crop region |
| `rotate` | Rotation angle |
| `flip` | Horizontal/vertical flip |

**Auto-format behavior (default: enabled):**

When auto-format is true (default), the system ignores the requested format and automatically selects: AVIF > WebP > JPEG/PNG based on browser capabilities.

To disable: append `?auto-format=false&format=jpeg` to the URL.

**Note:** Smart Imaging does not support AVIF for DM Prime customers (only DM Ultimate).

---

### 16. PDF and Document Delivery

**Animated GIFs:**

Use `is/content` instead of `is/image` to render animated GIFs:

```
<!-- Correct: preserves animation -->
https://myco.scene7.com/is/content/myco/animated-banner

<!-- Incorrect: strips animation, applies image processing -->
https://myco.scene7.com/is/image/myco/animated-banner
```

Image transformation commands do not apply when using `is/content`.

**PDF delivery:**

PDFs can be served via Dynamic Media URLs with `fmt=pdf`, or directly from AEM as standard DAM assets. For document viewers, consider the eCatalog viewer for multi-page PDF browsing.

---

### 17. Image Optimization Checklist

Apply this checklist to every AEM project for optimal image delivery:

**Format selection:**

- [ ] Smart Imaging enabled (auto AVIF/WebP negotiation)
- [ ] Do NOT hardcode `fmt=webp` or `fmt=avif` when Smart Imaging is active
- [ ] Use `png-alpha` or `webp-alpha` only when transparency is required
- [ ] SVGs served directly without rasterization for icons and illustrations

**Quality settings:**

- [ ] JPEG quality set to 80-85 (`qlt=85,0` as baseline)
- [ ] `resMode=sharp2` used for all resized images
- [ ] Unsharp mask applied: `op_usm=1.75,0.3,2,0` as starting point
- [ ] Quality never set above 90 (diminishing returns, excessive file size)

**Responsive delivery:**

- [ ] `srcset` with appropriate breakpoints configured (320, 640, 960, 1280, 1920)
- [ ] `sizes` attribute accurately reflects layout behavior
- [ ] `width` and `height` attributes set to prevent CLS
- [ ] Art-directed crops via `<picture>` element when aspect ratios change per breakpoint

**Loading strategy:**

- [ ] `loading="lazy"` on all below-fold images (default in Core Image Component)
- [ ] `loading="eager"` and `fetchpriority="high"` on the LCP image
- [ ] Preconnect hints for Dynamic Media CDN domains in `<head>`
- [ ] `preload` the LCP image in `<head>` for critical rendering path:

```html
<link rel="preload" as="image" href="https://myco.scene7.com/is/image/myco/hero?wid=1200&qlt=85"
      imagesrcset="https://myco.scene7.com/is/image/myco/hero?wid=640&qlt=85 640w,
                   https://myco.scene7.com/is/image/myco/hero?wid=1200&qlt=85 1200w,
                   https://myco.scene7.com/is/image/myco/hero?wid=1920&qlt=85 1920w"
      imagesizes="100vw" />
```

**CDN and caching:**

- [ ] Scene7 CDN TTL reviewed (default 10h; adjust if assets change frequently)
- [ ] Dispatcher cache headers set for `/content/dam` paths
- [ ] CDN invalidation template configured for post-publish cache clearing

---

### 18. Anti-Patterns

**Serving original assets directly:**

```html
<!-- WRONG: serves the 8MB original upload -->
<img src="/content/dam/mysite/photos/hero.jpg" alt="Hero" />

<!-- CORRECT: use Core Image Component or DM URL with size constraints -->
<img src="/content/dam/mysite/photos/hero.coreimg.85.1200.jpeg" alt="Hero" />
```

**Fixed dimensions for all viewports:**

```html
<!-- WRONG: 1920px image on all devices -->
<img src="https://myco.scene7.com/is/image/myco/banner?wid=1920" alt="Banner" />

<!-- CORRECT: responsive with srcset -->
<img src="https://myco.scene7.com/is/image/myco/banner?wid=1920"
     srcset="https://myco.scene7.com/is/image/myco/banner?wid=480 480w,
             https://myco.scene7.com/is/image/myco/banner?wid=960 960w,
             https://myco.scene7.com/is/image/myco/banner?wid=1920 1920w"
     sizes="100vw"
     alt="Banner" />
```

**Missing alt text:**

```html
<!-- WRONG: no alt text, accessibility violation -->
<img src="https://myco.scene7.com/is/image/myco/product" />

<!-- WRONG: non-descriptive alt text -->
<img src="https://myco.scene7.com/is/image/myco/product" alt="image" />

<!-- CORRECT: descriptive alt text from DAM metadata -->
<img src="https://myco.scene7.com/is/image/myco/product" alt="Red running shoe, side view, size 10" />
```

**No lazy loading on long pages:**

```html
<!-- WRONG: all 50 product images load immediately -->
<div class="product-grid">
    <img src="product-1.jpg" alt="Product 1" />
    <img src="product-2.jpg" alt="Product 2" />
    <!-- ... 48 more images ... -->
</div>

<!-- CORRECT: lazy load below-fold images -->
<div class="product-grid">
    <!-- First row visible on load -->
    <img src="product-1.jpg" alt="Product 1" loading="eager" />
    <img src="product-2.jpg" alt="Product 2" loading="eager" />
    <!-- Remaining rows lazy loaded -->
    <img src="product-3.jpg" alt="Product 3" loading="lazy" />
    <img src="product-4.jpg" alt="Product 4" loading="lazy" />
    <!-- ... -->
</div>
```

**Ignoring DPR (Device Pixel Ratio):**

```html
<!-- WRONG: assumes 1x DPR, blurry on Retina/HiDPI screens -->
<img src="https://myco.scene7.com/is/image/myco/logo?wid=200" alt="Logo"
     style="width:200px" />

<!-- CORRECT: let Smart Imaging handle DPR automatically, or provide 2x srcset -->
<img src="https://myco.scene7.com/is/image/myco/logo?wid=200"
     srcset="https://myco.scene7.com/is/image/myco/logo?wid=200 1x,
             https://myco.scene7.com/is/image/myco/logo?wid=400 2x"
     alt="Company logo"
     width="200" height="60" />
```

**Bypassing the Adaptive Image Servlet by linking DAM assets directly:**

```html
<!-- WRONG: bypasses all optimization -->
<img src="/content/dam/mysite/photo.jpg" alt="Photo" />

<!-- CORRECT: use the Core Image Component output format -->
<img src="/content/dam/mysite/photo.coreimg.85.960.jpeg/1680000000000/photo.jpeg" alt="Photo" />
```

---

### 19. Performance Summary: Cache Headers and Loading Strategy

**Dynamic Media URL cache headers:**

- Default Scene7 CDN TTL: 10 hours
- Configurable at: Tools > Assets > Dynamic Media Publish Setup > Image Serving > Common Thumbnail Attributes > Default cache time to live
- CDN invalidation flushes cache within minutes (vs. waiting for TTL expiration)
- Smart Imaging CDN: invalidating a base URL purges all variant URLs automatically

**Preconnect hints for third-party domains:**

```html
<!-- Add to customheaderlibs.html for every AEM page -->
<!-- Dynamic Media Scene7 -->
<link rel="preconnect" href="https://myco.scene7.com" crossorigin />

<!-- Dynamic Media with OpenAPI -->
<link rel="preconnect" href="https://delivery-pXXXX-eYYYY.adobeaemcloud.com" crossorigin />

<!-- Fonts or other third-party asset CDNs -->
<link rel="preconnect" href="https://use.typekit.net" crossorigin />
```

**Image loading strategy by position:**

| Image Position | loading | fetchpriority | Preload in head | Notes |
|---|---|---|---|---|
| LCP / Hero image | `eager` | `high` | Yes | Always preload the LCP image |
| Above-fold secondary | `eager` | `auto` | No | Visible on load but not LCP |
| Below-fold content | `lazy` | `auto` | No | Default Core Component behavior |
| Footer logos/icons | `lazy` | `low` | No | Low priority, rarely seen |

**Recommended baseline for Dynamic Media image URLs:**

```
fmt=jpg&qlt=85,0&resMode=sharp2&op_usm=1.75,0.3,2,0
```

This combination produces excellent results under most circumstances and should be used as the starting point for all image presets.
