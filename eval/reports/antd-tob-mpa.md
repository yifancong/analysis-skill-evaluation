## Summary
This MPA case is dominated by three avoidable sources of output bloat: bare side-effect imports, a barrel-file pattern that defeats tree-shaking, and oversized resource imports. The configuration also defines 14 entry pages without any shared-chunk extraction, so `react`, `react-dom`, `antd`, and the repeated feature pages are predisposed to duplicate code across outputs.

## Findings

### Finding 1: Five modules are imported purely for side effects (`Q1`)
- **Problem**: The page imports five modules as bare `import 'module'` statements and does not consume their exports, so they are kept only because of top-level side effects.
- **Evidence**: [`cases/antd-tob-mpa/src/pages/side-effects-only-imports/index.jsx`](./cases/antd-tob-mpa/src/pages/side-effects-only-imports/index.jsx) lines 13-17 import `register-polyfills`, `register-logger`, `register-error-handler`, `register-performance-marks`, and `register-theme`. Their effects are explicit:
  `register-polyfills` writes `globalThis.__POLYFILL_REGISTRY__` in [`cases/antd-tob-mpa/src/side-effects/register-polyfills.js`](./cases/antd-tob-mpa/src/side-effects/register-polyfills.js) lines 5-15;
  `register-logger` creates `globalThis.__APP_LOGGER__` in [`cases/antd-tob-mpa/src/side-effects/register-logger.js`](./cases/antd-tob-mpa/src/side-effects/register-logger.js) lines 7-32;
  `register-error-handler` attaches `window` listeners and writes `globalThis.__ERROR_BUFFER__` in [`cases/antd-tob-mpa/src/side-effects/register-error-handler.js`](./cases/antd-tob-mpa/src/side-effects/register-error-handler.js) lines 26-38;
  `register-performance-marks` records `performance.mark('app:init')` and exports helpers in [`cases/antd-tob-mpa/src/side-effects/register-performance-marks.js`](./cases/antd-tob-mpa/src/side-effects/register-performance-marks.js) lines 5-28;
  `register-theme` injects CSS variables through `document.documentElement.style.setProperty` in [`cases/antd-tob-mpa/src/side-effects/register-theme.js`](./cases/antd-tob-mpa/src/side-effects/register-theme.js) lines 5-31.
- **Impact**: These files cannot be removed by tree-shaking because the page explicitly depends on their side effects. They increase the page payload even when the exported helpers are unused.
- **Suggestion**: Move each side effect behind an explicit runtime call so the page imports only what it needs:

```js
// before
import '../../side-effects/register-theme';

// after
import { applyTheme } from '../../side-effects/theme';
applyTheme();
```

### Finding 2: The MPA config duplicates common framework code across 14 entrypoints
- **Problem**: The build defines 14 HTML entrypoints but does not configure `optimization.splitChunks`, so common dependencies are at risk of being bundled repeatedly.
- **Evidence**: [`cases/antd-tob-mpa/rspack.config.mjs`](./cases/antd-tob-mpa/rspack.config.mjs) lines 18-32 define 14 entries: `dashboard`, `charts`, `orders`, `product-detail`, `create-order`, `new-feature`, `feature-duplicate-1` through `feature-duplicate-5`, `side-effects-only-imports`, `esm-resolved-to-cjs`, and `large-assets`. The same config file contains no `optimization.splitChunks` block. All pages are React entrypoints and several use `antd`, `react`, and `react-dom`.
- **Impact**: Common framework/runtime code is emitted into many page bundles instead of being cached once. This is a structural output-optimization problem even before looking at page-specific waste.
- **Suggestion**: Extract shared framework chunks:

```js
export default {
  optimization: {
    splitChunks: {
      chunks: 'all',
      cacheGroups: {
        framework: {
          test: /[\\/]node_modules[\\/](react|react-dom|antd|@ant-design|rc-)[\\/]/,
          name: 'framework',
          priority: 30,
        },
      },
    },
  },
};
```

### Finding 3: `feature-duplicate-1` through `feature-duplicate-5` are near-identical pages
- **Problem**: Five entry pages duplicate the same component graph and differ only in labels.
- **Evidence**: [`cases/antd-tob-mpa/src/pages/feature-duplicate-1/index.jsx`](./cases/antd-tob-mpa/src/pages/feature-duplicate-1/index.jsx) and [`cases/antd-tob-mpa/src/pages/feature-duplicate-5/index.jsx`](./cases/antd-tob-mpa/src/pages/feature-duplicate-5/index.jsx) both import `AppLayout`, `PageHeader`, `OverviewCards`, `dashboardMetrics`, and `../../styles/global.css`, then render the same structure with only the title text changed.
- **Impact**: Each page is built as its own entry and repeats the same app code, producing avoidable duplication across outputs.
- **Suggestion**: Replace the copied pages with one parameterized page or a shared route-level module:

```js
function FeatureDuplicatePage({ index }) {
  return <PageHeader title={`功能副本 ${index}`} />;
}
```

### Finding 4: The barrel file pulls in three unused utility modules and defeats tree-shaking (`Q2` + `Q4`)
- **Problem**: The page only uses `formatDate` and `formatCurrency`, but it imports them from a barrel file that re-exports five submodules. Because the submodules carry module-level side effects, unused ones cannot be removed safely.
- **Evidence**: [`cases/antd-tob-mpa/src/pages/side-effects-only-imports/index.jsx`](./cases/antd-tob-mpa/src/pages/side-effects-only-imports/index.jsx) line 24 imports from `../../side-effects/utils`, and lines 20-24 explain that all five utility modules are dragged in. [`cases/antd-tob-mpa/src/side-effects/utils/index.js`](./cases/antd-tob-mpa/src/side-effects/utils/index.js) re-exports `date-utils`, `number-utils`, `string-utils`, `validator-utils`, and `color-utils` on lines 13-65. The page description explicitly calls out that `string-utils`, `validator-utils`, and `color-utils` are unused but still bundled.
- **Impact**: This is a direct tree-shaking failure. The page pays for three unused utility modules plus all of their exports and top-level side effects.
- **Suggestion**: Import the needed modules directly instead of through the barrel:

```js
import { formatDate } from '../../side-effects/utils/date-utils';
import { formatCurrency } from '../../side-effects/utils/number-utils';
```

### Finding 5: Large assets are bundled directly into the page output
- **Problem**: The page imports a large SVG, a large JSON sprite map, and CSS with base64-embedded fonts directly into the bundle.
- **Evidence**: [`cases/antd-tob-mpa/src/pages/large-assets/index.jsx`](./cases/antd-tob-mpa/src/pages/large-assets/index.jsx) imports `hero-banner.svg` on line 13, `resource-map.json` on line 18, and `enterprise-font.css` on line 23. The file documents the expected waste: about 190 KB SVG (lines 11-13, 49-53), about 108 KB JSON with 200 icons where only 3 are used (lines 15-18, 27-28, 65-69), and about 391 KB CSS containing base64 font data (lines 20-23, 102-106). The summary section totals the overhead at about 689 KB on lines 121-133.
- **Impact**: These resources bloat the initial payload, especially the JSON that gets serialized into JS and the font CSS that prevents independent caching.
- **Suggestion**: Emit the SVG and fonts as external files and fetch icon metadata on demand:

```js
const usedIcons = await Promise.all([
  import('../../assets/icons/icon-0001.json'),
  import('../../assets/icons/icon-0042.json'),
  import('../../assets/icons/icon-0100.json'),
]);
```

Use external `woff2` URLs in CSS:

```css
@font-face {
  font-family: 'Enterprise Sans';
  src: url('/fonts/enterprise-sans.woff2') format('woff2');
}
```

### Finding 6: The config turns off the `esm-resolved-to-cjs` rule even though that pattern hurts tree-shaking
- **Problem**: The build disables the Rsdoctor rule that would highlight ESM imports resolving to CJS, which is a known tree-shaking and scope-hoisting regression.
- **Evidence**: [`cases/antd-tob-mpa/rspack.config.mjs`](./cases/antd-tob-mpa/rspack.config.mjs) lines 79-85 configure `RsdoctorRspackPlugin` and explicitly set `'esm-resolved-to-cjs': 'off'`.
- **Impact**: If a dependency path resolves to CJS, the optimizer loses static export information and may retain more code than necessary. Turning the rule off hides that risk instead of fixing it.
- **Suggestion**: Re-enable the rule at least as `warn`, and then migrate any offending imports to packages or entrypoints that expose true ESM builds.

```js
linter: {
  rules: {
    'esm-resolved-to-cjs': 'warn',
  },
}
```

## Priority
- **High**: Finding 2 (14-entry MPA without shared-chunk extraction)
- **High**: Finding 4 (barrel file prevents tree-shaking and pulls unused modules)
- **High**: Finding 5 (large assets imported directly into output)
- **Medium**: Finding 1 (five side-effects-only modules)
- **Medium**: Finding 3 (five duplicated feature pages)
- **Medium**: Finding 6 (`esm-resolved-to-cjs` rule disabled)
