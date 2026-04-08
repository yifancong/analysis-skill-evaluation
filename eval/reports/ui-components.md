## Summary
This case deliberately concentrates three classes of bundle waste in one project: duplicate transitive packages across multiple UI ecosystems, cross-entry dependency duplication caused by `optimization.splitChunks: false`, and `E1006` module duplication where shared modules are pulled into both initial and async chunks. The bundle evidence confirms large entry chunks, duplicate `@emotion` subpackages, and repeated shared modules across `cross-chunks-*` and `module-mixed-chunks`.

## Findings

### Finding 1: Multiple UI frameworks create redundant package weight and duplicate transitive packages
- **Problem**: The project depends on many React and Vue UI ecosystems at the same time, which inflates the bundle baseline and also introduces duplicate transitive packages.
- **Evidence**: [`cases/ui-components/package.json`](./cases/ui-components/package.json) depends on `@mui/material`, `@chakra-ui/react`, `antd`, `react-bootstrap`, `@mantine/core`, `@fluentui/react`, `@headlessui/react`, `element-plus`, `vuetify`, `vant`, and `vue`. [`cases/ui-components/src/entries/duplicate-packages.js`](./cases/ui-components/src/entries/duplicate-packages.js) explains the conflicting chains: `antd -> @ant-design/cssinjs -> @emotion/hash@0.8.0` and `@emotion/unitless@0.7.5`, while `@mui/material` and `@chakra-ui/react` resolve `@emotion/hash@0.9.2` and `@emotion/unitless@0.10.0`. Flattened Rsdoctor package data confirms both versions of `@emotion/hash` and `@emotion/unitless` are present in the bundle.
- **Impact**: More framework code is shipped than any single entry needs, and the same `@emotion` subpackages are bundled twice under different versions. That raises parse cost and makes long-term caching worse.
- **Suggestion**: Remove unused UI ecosystems where possible, and unify the duplicate `@emotion` subpackages. For build-side unification:

```js
import path from 'node:path';

export default {
  resolve: {
    alias: {
      '@emotion/hash': path.resolve('node_modules/@emotion/hash'),
      '@emotion/unitless': path.resolve('node_modules/@emotion/unitless'),
    },
  },
};
```

For package-manager-side unification:

```json
{
  "pnpm": {
    "overrides": {
      "@emotion/hash": "0.9.2",
      "@emotion/unitless": "0.10.0"
    }
  }
}
```

### Finding 2: `splitChunks: false` duplicates shared `antd` and `@mui/material` code across entry chunks
- **Problem**: `cross-chunks-a` and `cross-chunks-b` both import large overlapping surfaces from `antd` and `@mui/material`, but the build disables shared-chunk extraction entirely.
- **Evidence**: [`cases/ui-components/rspack.config.mjs`](./cases/ui-components/rspack.config.mjs) sets `optimization.splitChunks: false`. [`cases/ui-components/src/entries/cross-chunks-a.js`](./cases/ui-components/src/entries/cross-chunks-a.js) and [`cases/ui-components/src/entries/cross-chunks-b.js`](./cases/ui-components/src/entries/cross-chunks-b.js) both import `Button`, `Input`, `Select`, `Table`, `Form`, and `Card` from `antd`, plus overlapping `@mui/material` components such as `Button`, `TextField`, `Select`, `Table`, and `Card`. The built assets are large for entry chunks that should be sharing framework code: `cross-chunks-a.js` is about 1.22 MB and `cross-chunks-b.js` is about 1.52 MB in [`cases/ui-components/dist`](./cases/ui-components/dist).
- **Impact**: The same framework modules are emitted into multiple entry bundles, which is the `E1002` pattern. Users pay repeated download and parse costs when navigating between entries.
- **Suggestion**: Re-enable `splitChunks` and extract vendor groups for shared frameworks:

```js
export default {
  optimization: {
    splitChunks: {
      chunks: 'all',
      cacheGroups: {
        antd: {
          test: /[\\/]node_modules[\\/](antd|@ant-design|rc-)/,
          name: 'vendor-antd',
          priority: 30,
        },
        mui: {
          test: /[\\/]node_modules[\\/](@mui|@emotion)[\\/]/,
          name: 'vendor-mui',
          priority: 25,
        },
      },
    },
  },
};
```

### Finding 3: Shared modules are duplicated in both initial and async chunks (`E1006`)
- **Problem**: The same user modules are imported synchronously by the entry and again through lazy pages, while `splitChunks` is disabled.
- **Evidence**: [`cases/ui-components/src/entries/module-mixed-chunks.js`](./cases/ui-components/src/entries/module-mixed-chunks.js) synchronously imports `../shared/analytics`, `../shared/chart-helpers`, and `../shared/user-info` on lines 19-21, then dynamically imports `./lazy-pages/lazy-analytics`, `./lazy-pages/lazy-charts`, and `./lazy-pages/lazy-user` on lines 48-66. The inline comments state that each lazy page also imports the same shared module, which is exactly the `E1006` condition. The shared modules themselves are defined in [`cases/ui-components/src/shared/analytics.js`](./cases/ui-components/src/shared/analytics.js), [`cases/ui-components/src/shared/chart-helpers.js`](./cases/ui-components/src/shared/chart-helpers.js), and [`cases/ui-components/src/shared/user-info.js`](./cases/ui-components/src/shared/user-info.js).
- **Impact**: `shared/analytics.js`, `shared/chart-helpers.js`, and `shared/user-info.js` can be copied into both the initial `module-mixed-chunks` entry and the async chunks, increasing payload size and invalidating cache locality for code that should exist once.
- **Suggestion**: Let shared user modules be extracted automatically or avoid importing them from both paths. For example:

```js
export default {
  optimization: {
    splitChunks: {
      chunks: 'all',
      minChunks: 2,
    },
  },
};
```

Or restructure so the shared logic is only loaded from the async boundary:

```js
async function loadAnalyticsPanel() {
  const { renderAnalyticsDashboard, calculateMetrics } = await import('../shared/analytics');
  return renderAnalyticsDashboard(calculateMetrics(data));
}
```

### Finding 4: Tree-shaking is materially weakened by the amount of side-effectful framework code being imported
- **Problem**: This entry set imports broad component surfaces from multiple UI libraries, and the flattened Rsdoctor data shows thousands of side-effectful modules across framework packages.
- **Evidence**: The flattened Rsdoctor side-effects report for `ui-components` shows 5,058 modules with bailout reasons, including 5,039 under `node_modules`, with large concentrations in `element-plus`, `antd`, `@mui/material`, and related packages. The shared modules also re-export concrete UI components from `antd` and `@mui/material`, such as [`cases/ui-components/src/shared/analytics.js`](./cases/ui-components/src/shared/analytics.js) lines 11-18 and [`cases/ui-components/src/shared/chart-helpers.js`](./cases/ui-components/src/shared/chart-helpers.js) lines 6-25.
- **Impact**: Even when individual call sites look small, importing from many heavy component packages reduces the optimizer’s ability to trim unused code and inflates both initial and async chunks.
- **Suggestion**: Narrow imports to the framework actually needed per entry and isolate framework-specific shared code. If an entry only needs data formatting, keep that logic out of a module that also imports UI components.

```js
// before
import { renderChartTable, formatChartData } from '../shared/chart-helpers';

// after
import { formatChartData } from '../shared/chart-data';
const { renderChartTable } = await import('../shared/chart-table');
```

## Priority
- **High**: Finding 2 (`E1002` cross-chunk duplication from `splitChunks: false`)
- **High**: Finding 3 (`E1006` shared modules duplicated across initial and async chunks)
- **Medium**: Finding 1 (multi-framework dependency redundancy and duplicate `@emotion` subpackages)
- **Medium**: Finding 4 (tree-shaking erosion from broad, side-effectful framework imports)
