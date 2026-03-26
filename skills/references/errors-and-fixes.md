# Common Errors & Fixes

Errors encountered during webpack 3 → 5 migration with their root causes and resolutions.

---

## Table of Contents
1. [Module not found: Can't resolve 'crypto'](#crypto)
2. [Can't resolve 'process'](#process)
3. [CommonsChunkPlugin is not a constructor](#commons-chunk)
4. [ExtractTextPlugin is not a constructor](#extract-text)
5. [Rule.loaders has been removed](#rule-loaders)
6. [Error: Cannot find module 'webpack/bin/config-yargs'](#config-yargs)
7. [HMR / Fast Refresh not working](#hmr)
8. [CSS Modules: class names changed](#css-modules)
9. [[contenthash] vs [hash] confusion](#hash)
10. [ReactDOM.render is no longer supported](#reactdom-render)
11. [devServer proxy format changed](#proxy)
12. [output.publicPath auto vs explicit](#publicpath)
13. [Chunk loading failed](#chunk-loading)
14. [SVG import broken](#svg)
15. [TypeScript: isolatedModules errors](#typescript)

---

## 1. Module not found: Can't resolve 'crypto' <a name="crypto"></a>

**Error:**
```
Module not found: Error: Can't resolve 'crypto' in '...'
```

**Cause:** webpack 5 removed automatic Node.js core module polyfills.

**Fix:**
```bash
npm install --save-dev crypto-browserify
```
```js
// webpack.config.js
resolve: {
  fallback: {
    crypto: require.resolve('crypto-browserify'),
  },
},
```

**Best practice:** Check if you actually need `crypto`. If it's only used server-side, move that code out of the browser bundle entirely.

---

## 2. Can't resolve 'process' / 'buffer' / 'stream' <a name="process"></a>

**Error:**
```
Module not found: Error: Can't resolve 'process/browser'
```

**Fix:**
```bash
npm install --save-dev process buffer stream-browserify
```
```js
const { ProvidePlugin } = require('webpack');

// webpack.config.js
plugins: [
  new ProvidePlugin({
    process: 'process/browser',
    Buffer: ['buffer', 'Buffer'],
  }),
],
resolve: {
  fallback: {
    process: require.resolve('process/browser'),
    buffer: require.resolve('buffer/'),
    stream: require.resolve('stream-browserify'),
  },
},
```

---

## 3. CommonsChunkPlugin is not a constructor <a name="commons-chunk"></a>

**Error:**
```
TypeError: webpack.optimize.CommonsChunkPlugin is not a constructor
```

**Cause:** `CommonsChunkPlugin` was removed in webpack 4 and does not exist in v5.

**Fix:** Replace with `optimization.splitChunks`:

```js
// REMOVE this entirely:
new webpack.optimize.CommonsChunkPlugin({ name: 'vendor', minChunks: ... })

// REPLACE in optimization block:
optimization: {
  splitChunks: {
    chunks: 'all',
    cacheGroups: {
      vendors: {
        test: /[\\/]node_modules[\\/]/,
        name: 'vendors',
        chunks: 'all',
      },
    },
  },
},
```

---

## 4. ExtractTextPlugin is not a constructor <a name="extract-text"></a>

**Error:**
```
TypeError: ExtractTextPlugin is not a constructor
```
or
```
The "extract-text-webpack-plugin" package requires webpack <= 4
```

**Fix:** Replace with `mini-css-extract-plugin`:

```bash
npm uninstall extract-text-webpack-plugin
npm install --save-dev mini-css-extract-plugin
```

```js
// REMOVE:
const ExtractTextPlugin = require('extract-text-webpack-plugin');
// ...
new ExtractTextPlugin('styles.[contenthash].css')
// ...
ExtractTextPlugin.extract({ fallback: 'style-loader', use: 'css-loader' })

// REPLACE:
const MiniCssExtractPlugin = require('mini-css-extract-plugin');
// ...plugins:
new MiniCssExtractPlugin({ filename: 'css/[name].[contenthash:8].css' })
// ...rules:
{ loader: MiniCssExtractPlugin.loader }
```

---

## 5. Rule.loaders has been removed <a name="rule-loaders"></a>

**Error:**
```
ValidationError: Rule.loaders has been removed (use Rule.use instead)
```

**Fix:** Replace `loaders` → `use` in all rules:

```js
// OLD
{ test: /\.js$/, loaders: ['babel-loader'] }

// NEW
{ test: /\.js$/, use: ['babel-loader'] }
// or simply
{ test: /\.js$/, loader: 'babel-loader' }
```

Also rename `module.loaders` → `module.rules` if still using the v1 API.

---

## 6. Cannot find module 'webpack/bin/config-yargs' <a name="config-yargs"></a>

**Error:**
```
Error: Cannot find module 'webpack/bin/config-yargs'
```

**Cause:** Using `webpack-dev-server` v3 CLI with webpack 5. The internal API changed.

**Fix:** Upgrade to webpack-dev-server v5:
```bash
npm uninstall webpack-dev-server
npm install --save-dev webpack-dev-server@5
```

Update your `start` script:
```json
"start": "webpack serve --config webpack.dev.js"
```
Not `webpack-dev-server` — it's now `webpack serve`.

---

## 7. HMR / Fast Refresh not working <a name="hmr"></a>

**Symptoms:** Changes don't reflect without full page reload; `react-hot-loader` errors.

**Cause:** `react-hot-loader` is not compatible with React 18.

**Fix:** Replace with React Fast Refresh:

```bash
npm uninstall react-hot-loader
npm install --save-dev @pmmmwh/react-refresh-webpack-plugin react-refresh
```

Remove all `import { hot } from 'react-hot-loader/root'` from components.

```js
// webpack.dev.js
const ReactRefreshPlugin = require('@pmmmwh/react-refresh-webpack-plugin');
// ...
plugins: [new ReactRefreshPlugin()],

// babel config — dev only:
plugins: [
  ...(process.env.NODE_ENV === 'development' ? ['react-refresh/babel'] : [])
]
```

---

## 8. CSS Modules: class names changed in production <a name="css-modules"></a>

**Symptom:** Styles break after build — class names in HTML don't match class names in CSS.

**Cause:** `css-loader` v6 changed the default `localIdentName` format.

**Fix:** Explicitly set `localIdentName` to match your old format OR update snapshots:

```js
// Dev: human-readable
{ modules: { auto: true, localIdentName: '[name]__[local]--[hash:base64:5]' } }

// Prod: short hashes
{ modules: { auto: true, localIdentName: '[hash:base64:8]' } }
```

---

## 9. [contenthash] vs [hash] in filenames <a name="hash"></a>

**Symptom:** All files get new hashes on every build, defeating long-term caching.

**Cause:** `[hash]` in webpack 5 is the build hash (whole build), not per-file. Always use `[contenthash]`.

**Fix:**
```js
output: {
  filename: '[name].[contenthash:8].js',        // ✓
  chunkFilename: '[name].[contenthash:8].chunk.js',  // ✓
  // NOT: '[name].[hash].js'                          ✗
}
```

Also add `optimization.runtimeChunk: 'single'` to prevent runtime changes from invalidating vendor hashes.

---

## 10. ReactDOM.render is no longer supported <a name="reactdom-render"></a>

**Warning/Error:**
```
Warning: ReactDOM.render is no longer supported in React 18.
Use createRoot instead.
```

**Fix:** Update `src/index.jsx` (or `src/index.js`):

```jsx
// REMOVE:
import ReactDOM from 'react-dom';
ReactDOM.render(<App />, document.getElementById('root'));

// ADD:
import { createRoot } from 'react-dom/client';
const container = document.getElementById('root');
const root = createRoot(container);
root.render(<App />);
```

---

## 11. devServer proxy format changed <a name="proxy"></a>

**Error:**
```
[webpack-dev-server] Invalid options object. webpack-dev-server has been initialized using an options object that does not match the API schema.
```

**Cause:** `proxy` in webpack-dev-server v5 must be an array, not an object.

**Fix:**
```js
// OLD v3:
proxy: {
  '/api': { target: 'http://localhost:8080', changeOrigin: true },
}

// NEW v5:
proxy: [
  { context: ['/api'], target: 'http://localhost:8080', changeOrigin: true },
]
```

---

## 12. output.publicPath issues <a name="publicpath"></a>

**Symptom:** Assets load in dev but return 404 in production, or vice versa.

**Fix options:**

```js
// Option A: absolute path for a known CDN/base URL
output: { publicPath: 'https://cdn.example.com/assets/' }

// Option B: relative paths (suitable for most SPAs)
output: { publicPath: '/' }

// Option C: auto (webpack 5 new default — infers at runtime)
output: { publicPath: 'auto' }
```

`'auto'` works well for most SPAs but can break when served from a sub-path. If assets 404 in prod, switch to an explicit `/` or your CDN base URL.

---

## 13. Chunk loading failed <a name="chunk-loading"></a>

**Error:**
```
ChunkLoadError: Loading chunk X failed.
```

**Possible causes and fixes:**

1. **Wrong `publicPath`** → see §12 above.
2. **`output.clean: true` removing old chunks while users have cached HTML** → use immutable cache headers + versioned filenames.
3. **Aggressive code splitting** → too many chunks causes race conditions on slow connections. Increase `minSize` in `splitChunks`:
   ```js
   splitChunks: { minSize: 30000 }
   ```
4. **Circular dependencies** → install and run `circular-dependency-plugin` to detect them:
   ```bash
   npm install --save-dev circular-dependency-plugin
   ```

---

## 14. SVG import broken <a name="svg"></a>

**Error:**
```
Module parse failed: Unexpected token (1:0) — you may need an appropriate loader
```

**Cause:** webpack 5 Asset Modules don't handle SVG as React components by default.

**Fix (using @svgr/webpack):**
```bash
npm install --save-dev @svgr/webpack
```

```js
{
  test: /\.svg$/i,
  issuer: /\.[jt]sx?$/,
  use: [{ loader: '@svgr/webpack' }],
},
// OR for both URL and React component import:
{
  test: /\.svg$/i,
  type: 'asset/inline',   // use as URL
},
```

Usage:
```jsx
import { ReactComponent as Logo } from './logo.svg';   // as component
import logoUrl from './logo.svg';                       // as URL
```

---

## 15. TypeScript: isolatedModules errors <a name="typescript"></a>

**Error (ts-loader or babel + TypeScript):**
```
Re-exporting a type when the '--isolatedModules' flag is provided requires using 'export type'
```

**Fix:** Update type exports:
```ts
// OLD
export { MyType } from './types';

// NEW
export type { MyType } from './types';
```

Alternatively, add `"isolatedModules": true` to `tsconfig.json` (needed for babel transpilation) and fix all errors it reveals — this is recommended regardless as it improves build performance.
