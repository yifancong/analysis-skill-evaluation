## Summary
This SPA already uses route-level lazy loading, but the output is still dominated by framework/runtime code and large route chunks. The successful build produced a 729 KB `main.js` plus several 400-880 KB async chunks, and the dependency set includes `react`, `react-dom`, `react-router-dom`, and `@iconify` packages without any explicit vendor-chunk strategy in the Rspack config.

## Findings

### Finding 1: The build lacks an explicit vendor split for React framework dependencies
- **Problem**: The app uses `react`, `react-dom`, and `react-router-dom`, but the config does not define any `splitChunks` strategy to isolate long-lived framework code.
- **Evidence**: [`cases/react-10k/src/index.jsx`](./cases/react-10k/src/index.jsx) imports `react-router-dom` on line 3 and bootstraps React on lines 16-33. [`cases/react-10k/package.json`](./cases/react-10k/package.json) lists `react`, `react-dom`, and `react-router-dom` in dependencies on lines 19-24. [`cases/react-10k/rspack.config.mjs`](./cases/react-10k/rspack.config.mjs) contains no `optimization.splitChunks` block.
- **Impact**: Framework/runtime code is more likely to stay coupled to application chunks, reducing cache reuse and contributing to the 729 KB `main.js` emitted in [`cases/react-10k/dist`](./cases/react-10k/dist).
- **Suggestion**: Extract the framework into a stable vendor chunk:

```js
export default {
  optimization: {
    splitChunks: {
      chunks: 'all',
      cacheGroups: {
        framework: {
          test: /[\\/]node_modules[\\/](react|react-dom|react-router-dom)[\\/]/,
          name: 'framework',
          priority: 30,
        },
      },
    },
  },
};
```

### Finding 2: `@iconify-icons/material-symbols` is a likely oversized dependency surface
- **Problem**: The project depends on the full `@iconify-icons/material-symbols` package, and many source files import icons from it.
- **Evidence**: [`cases/react-10k/package.json`](./cases/react-10k/package.json) lines 19-21 include `@iconify-icons/material-symbols` and `@iconify/react`. The source tree imports many individual Material Symbols files, for example `cases/react-10k/src/f0.jsx`, `cases/react-10k/src/f1.jsx`, and many nested `d*/f*.jsx` files reported by ripgrep under [`cases/react-10k/src`](./cases/react-10k/src).
- **Impact**: Even if each icon file is individually imported, the icon system still contributes meaningful weight across many route chunks. This is consistent with the build warning output and the large emitted assets such as `113.js` at 881 KB and several 436-460 KB route chunks in [`cases/react-10k/dist`](./cases/react-10k/dist).
- **Suggestion**: Audit icon usage and switch to a lighter icon strategy where possible. If only a small set is required, keep imports tightly scoped and consider a smaller icon package or SVG sprite pipeline.

```js
// Keep imports as specific as possible
import phoneAndroidSharp from '@iconify-icons/material-symbols/phone-android-sharp.js';
```

If only a handful of icons are needed globally, replace the package with hand-authored SVG components.

## Priority
- **High**: Finding 1 (missing vendor chunk for `react` / `react-dom` / `react-router-dom`)
- **Medium**: Finding 2 (`@iconify-icons/material-symbols` likely contributes unnecessary weight)
