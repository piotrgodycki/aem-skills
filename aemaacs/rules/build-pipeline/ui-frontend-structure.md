---
title: ui.frontend Module Structure & Build Pipeline
impact: HIGH
impactDescription: Correct module structure prevents build failures and enables frontend pipeline deployment
tags: webpack, scss, typescript, build, ui.frontend, clientlib-generator, frontend-pipeline
---

## ui.frontend Module Structure & Build Pipeline

The `ui.frontend` module is the central location for all frontend resources. It uses Webpack to bundle JS/CSS/SCSS/TS and `aem-clientlib-generator` to deploy compiled assets into `ui.apps` ClientLibs.

---

### 1. Folder Structure

```
ui.frontend/
├── src/main/webpack/
│   ├── components/          # Per-component JS/CSS
│   │   ├── _hero.js
│   │   ├── _hero.scss
│   │   ├── _tabs.js
│   │   └── _tabs.scss
│   ├── resources/           # Images, fonts, icons
│   │   ├── fonts/
│   │   └── images/
│   ├── site/
│   │   ├── _variables.scss  # SCSS variables (brand colors, spacing)
│   │   ├── _mixins.scss     # SCSS mixins (breakpoints, typography)
│   │   ├── _base.scss       # Reset / global styles
│   │   ├── main.scss        # Main SCSS entry (imports all partials)
│   │   └── main.ts          # Main JS/TS entry
│   └── static/
│       └── index.html       # Dev server template
├── dist/
│   ├── clientlib-site/      # Compiled site assets → ui.apps
│   └── clientlib-dependencies/ # Third-party vendor assets
├── webpack.common.js        # Shared config (entry, output, loaders)
├── webpack.dev.js           # Dev server, source maps, proxy
├── webpack.prod.js          # Minification, tree shaking, optimization
├── clientlib.config.js      # aem-clientlib-generator mapping
├── tsconfig.json            # TypeScript configuration
└── package.json             # Dependencies and scripts
```

---

### 2. Build Flow

```
Developer edits src/main/webpack/
  ↓
Webpack bundles → dist/clientlib-site/
  ↓
aem-clientlib-generator copies → ui.apps/.../clientlibs/clientlib-site/
  ↓
Maven builds content package → installs to AEM
  mvn clean install -PautoInstallSinglePackage
```

### Build Commands

```bash
npm run dev      # Full build, no optimization, source maps enabled
npm run prod     # Full build, minification, tree shaking, no source maps
npm run start    # webpack-dev-server at localhost:8080 with proxy to AEM
npm run lint     # ESLint + Stylelint
npm run test     # Jest unit tests
```

---

### 3. Webpack Configuration

#### webpack.common.js — Shared Config

```javascript
const path = require('path');
const MiniCssExtractPlugin = require('mini-css-extract-plugin');
const CopyWebpackPlugin = require('copy-webpack-plugin');

module.exports = {
    entry: {
        site: './src/main/webpack/site/main.ts',
    },
    output: {
        filename: 'clientlib-site/[name].js',
        path: path.resolve(__dirname, 'dist'),
    },
    module: {
        rules: [
            // TypeScript
            {
                test: /\.tsx?$/,
                exclude: /node_modules/,
                use: ['ts-loader'],
            },
            // SCSS
            {
                test: /\.scss$/,
                use: [
                    MiniCssExtractPlugin.loader,
                    { loader: 'css-loader', options: { url: false } },
                    'postcss-loader', // autoprefixer, etc.
                    'sass-loader',
                ],
            },
            // Fonts and images in CSS
            {
                test: /\.(woff2?|ttf|eot)$/,
                type: 'asset/resource',
                generator: { filename: 'clientlib-site/resources/[name][ext]' },
            },
        ],
    },
    plugins: [
        new MiniCssExtractPlugin({ filename: 'clientlib-site/[name].css' }),
        new CopyWebpackPlugin({
            patterns: [
                { from: 'src/main/webpack/resources/', to: 'clientlib-site/resources/' },
            ],
        }),
    ],
    resolve: {
        extensions: ['.ts', '.tsx', '.js'],
    },
    // Auto-discover component files (glob import)
    stats: {
        children: true,
    },
};
```

#### webpack.dev.js — Development

```javascript
const { merge } = require('webpack-merge');
const common = require('./webpack.common.js');

module.exports = merge(common, {
    mode: 'development',
    devtool: 'inline-source-map', // Full source maps for debugging
    performance: { hints: 'warning' },
    devServer: {
        proxy: [{
            context: ['/content', '/etc.clientlibs', '/libs'],
            target: 'http://localhost:4502', // AEM author
            auth: 'admin:admin',
            changeOrigin: true,
        }],
        client: { overlay: { errors: true, warnings: false } },
        hot: true, // Hot Module Replacement
        port: 8080,
    },
});
```

#### webpack.prod.js — Production

```javascript
const { merge } = require('webpack-merge');
const common = require('./webpack.common.js');
const TerserPlugin = require('terser-webpack-plugin');
const CssMinimizerPlugin = require('css-minimizer-webpack-plugin');

module.exports = merge(common, {
    mode: 'production',
    devtool: false, // No source maps in production
    optimization: {
        minimize: true,
        minimizer: [
            new TerserPlugin({
                terserOptions: {
                    compress: { drop_console: true }, // Remove console.log
                    output: { comments: false },
                },
            }),
            new CssMinimizerPlugin(),
        ],
        // Code splitting for vendor dependencies
        splitChunks: {
            cacheGroups: {
                vendor: {
                    test: /[\\/]node_modules[\\/]/,
                    name: 'clientlib-dependencies/vendor',
                    chunks: 'all',
                },
            },
        },
    },
});
```

---

### 4. ClientLib Generator Configuration

```javascript
// clientlib.config.js
module.exports = {
    context: __dirname,
    clientLibRoot: '../ui.apps/src/main/content/jcr_root/apps/myproject/clientlibs',
    libs: [
        {
            name: 'clientlib-site',
            categories: ['myproject.site'],
            serializationFormat: 'xml',
            assets: {
                js: ['dist/clientlib-site/site.js'],
                css: ['dist/clientlib-site/site.css'],
                resources: ['dist/clientlib-site/resources/*'],
            },
        },
        {
            name: 'clientlib-dependencies',
            categories: ['myproject.dependencies'],
            serializationFormat: 'xml',
            embed: [],
            assets: {
                js: ['dist/clientlib-dependencies/*.js'],
                css: ['dist/clientlib-dependencies/*.css'],
            },
        },
    ],
};
```

---

### 5. SCSS Organization

```scss
// site/main.scss — entry point
@import 'variables';      // Brand tokens: colors, spacing, typography
@import 'mixins';         // Breakpoints, typography helpers
@import 'base';           // Reset, global styles, HTML elements

// Auto-import all component SCSS files
// Webpack globbing discovers these automatically
@import '../components/**/*.scss';
```

```scss
// site/_variables.scss
$brand-primary: #0066cc;
$brand-secondary: #004080;
$font-family-base: 'Source Sans Pro', Helvetica, Arial, sans-serif;
$font-family-heading: 'Asar', serif;

$spacing-xs: 4px;
$spacing-sm: 8px;
$spacing-md: 16px;
$spacing-lg: 32px;
$spacing-xl: 64px;

// Breakpoints (match AEM responsive grid)
$breakpoint-phone: 768px;
$breakpoint-tablet: 1200px;
```

```scss
// site/_mixins.scss
@mixin respond-to($breakpoint) {
  @if $breakpoint == phone {
    @media (max-width: $breakpoint-phone) { @content; }
  } @else if $breakpoint == tablet {
    @media (max-width: $breakpoint-tablet) { @content; }
  }
}
```

---

### 6. TypeScript Integration

```json
// tsconfig.json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "ESNext",
    "moduleResolution": "node",
    "strict": true,
    "esModuleInterop": true,
    "declaration": false,
    "sourceMap": true,
    "outDir": "./dist",
    "baseUrl": "./src/main/webpack",
    "paths": {
      "@components/*": ["components/*"],
      "@site/*": ["site/*"]
    }
  },
  "include": ["src/main/webpack/**/*.ts"],
  "exclude": ["node_modules", "dist"]
}
```

---

### 7. Frontend Pipeline vs Full-Stack Pipeline

| Feature | Full-Stack Pipeline | Frontend-Only Pipeline |
|---------|-------------------|----------------------|
| What deploys | Everything (Java, content, frontend) | Only `ui.frontend` artifacts |
| Build tool | Maven → npm (via frontend-maven-plugin) | npm only |
| Delivery | via `/etc.clientlibs/` | Direct CDN delivery |
| Deploy time | 30-60 minutes | 5-15 minutes |
| Use case | Java/content changes | CSS/JS-only changes |

**Frontend pipeline requirements:**
1. Project must use the AEM archetype's `ui.frontend` module structure
2. Site theme configured in Cloud Manager
3. `package.json` must have `build` script

---

### 8. Local Development with Proxy

```javascript
// webpack.dev.js — proxy config for local AEM
devServer: {
    proxy: [{
        context: ['/content', '/etc.clientlibs', '/libs', '/api'],
        target: 'http://localhost:4502',
        auth: 'admin:admin',        // Local dev credentials
        changeOrigin: true,
        secure: false,
    }],
    hot: true,
    liveReload: true,
    watchFiles: ['src/**/*'],
}
```

**Workflow:**
1. Start AEM SDK: `java -jar aem-sdk-quickstart.jar`
2. Start Webpack dev server: `npm run start`
3. Open `http://localhost:8080/content/mysite/en.html`
4. Edit SCSS/TS files → HMR updates in browser instantly

---

### 9. Environment-Specific Builds

```javascript
// webpack.common.js — environment variables
const webpack = require('webpack');

plugins: [
    new webpack.DefinePlugin({
        'process.env.AEM_ENV': JSON.stringify(process.env.AEM_ENV || 'local'),
        'process.env.ANALYTICS_ID': JSON.stringify(process.env.ANALYTICS_ID || ''),
    }),
],
```

---

### 10. Anti-Patterns

#### Editing Generated Output

```bash
# WRONG — these are overwritten on build
vi ui.frontend/dist/clientlib-site/site.css
vi ui.apps/.../clientlibs/clientlib-site/css.txt

# CORRECT — edit source files
vi ui.frontend/src/main/webpack/components/_hero.scss
vi ui.frontend/src/main/webpack/site/main.ts
```

#### Skipping the ClientLib Generator

```bash
# WRONG — manually copying files to ui.apps
cp dist/clientlib-site/site.css ../ui.apps/.../clientlibs/

# CORRECT — let aem-clientlib-generator handle it
npm run prod  # builds and copies via clientlib.config.js
```

#### Mixing Module Systems

```javascript
// WRONG — mixing CommonJS and ES modules
const utils = require('./utils');  // CommonJS
import { helper } from './helper'; // ES module

// CORRECT — use ES modules consistently
import { utils } from './utils';
import { helper } from './helper';
```

#### Not Tree Shaking

```javascript
// WRONG — importing entire library
import _ from 'lodash'; // 70KB+ bundled

// CORRECT — import only what you need
import debounce from 'lodash/debounce'; // ~1KB
```

#### Missing postcss.config.js

```javascript
// postcss.config.js — required for autoprefixer
module.exports = {
  plugins: {
    autoprefixer: {}, // Adds vendor prefixes automatically
  },
};
// Without this, CSS may not work in older browsers
```
