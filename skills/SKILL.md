---
name: webpack-migration
description: >
  Step-by-step guide for migrating an existing webpack 3 project to webpack 5,
  specifically targeting React 18 applications. Use this skill whenever the user
  mentions migrating, upgrading, or converting a webpack project — especially when
  keywords like "webpack 3 to 5", "webpack upgrade", "webpack migration", "modernize
  build", or "optimize webpack build times" appear. Also trigger when the user shows
  a webpack.config.js with v3-era patterns (ExtractTextPlugin, CommonsChunkPlugin,
  UglifyJsPlugin, loaders array, etc.) and wants to update it. This skill covers
  dependency updates, config rewrites, React 18 compatibility, performance
  optimizations, and common breaking-change fixes. Always use this skill rather than
  guessing — webpack 3→5 has many subtle breaking changes that require precise guidance.
---

# Webpack 3 → 5 Migration Guide (React 18)

> **Scope**: Migrating an *existing* production project. Not a new-project setup.
> **Target**: webpack 5 latest + React 18 + modern tooling best practices.

---

## How to Use This Skill

1. **Read this file first** — it gives you the full migration workflow.
2. **Load reference files on demand** (see §References at the bottom).
   - Deep config examples → `references/config-examples.md`
   - Common errors & fixes → `references/errors-and-fixes.md`
   - Performance tuning → `references/performance.md`

Always audit the user's existing `webpack.config.js`, `package.json`, and (if present) `babel.config.js` / `.babelrc` before prescribing changes. Ask the user to paste them if not already shown.

---

## Phase 0 — Pre-Migration Audit

Before touching anything, gather:

```
1. Current webpack version        → package.json "webpack"
2. Current Node.js version        → node -v  (webpack 5 needs Node ≥ 12.13, recommend ≥ 18)
3. Bundler plugins in use         → list every devDependency containing "webpack-"
4. Loaders in use                 → list every devDependency ending in "-loader"
5. Babel preset/plugins           → .babelrc or babel.config.js
6. CSS approach                   → CSS Modules? Sass? PostCSS? Styled-components?
7. Code splitting strategy        → CommonsChunkPlugin? DllPlugin? dynamic import()?
8. Output targets                 → browserslist? targets in babel config?
9. Source maps in use             → devtool value
10. Environment variable strategy → DefinePlugin? dotenv-webpack?
```

Create a **migration branch** before starting. Never migrate on main.

---

## Phase 1 — Upgrade Node & Core Dependencies

### 1.1 Node.js
webpack 5 officially requires Node ≥ 12.13.0. For best performance use Node 18 LTS or 20 LTS (enables native ES module support and faster crypto).

```bash
# Check current version
node -v

# Upgrade via nvm (recommended)
nvm install 20
nvm use 20
```

### 1.2 Core webpack packages

Remove old packages, install new ones in one shot to avoid peer-dep conflicts:

```bash
# Remove everything webpack-related first
npm uninstall webpack webpack-cli webpack-dev-server \
  webpack-merge extract-text-webpack-plugin \
  html-webpack-plugin uglifyjs-webpack-plugin \
  optimize-css-assets-webpack-plugin \
  hard-source-webpack-plugin \
  babel-loader babel-core \
  css-loader style-loader sass-loader \
  file-loader url-loader

# Install webpack 5 core
npm install --save-dev \
  webpack@5 \
  webpack-cli@5 \
  webpack-dev-server@5 \
  webpack-merge@5

# Install modern replacements
npm install --save-dev \
  html-webpack-plugin@5 \
  mini-css-extract-plugin@2 \
  css-minimizer-webpack-plugin@5 \
  terser-webpack-plugin@5 \
  babel-loader@9 \
  css-loader@6 \
  style-loader@3 \
  sass-loader@13 \
  postcss-loader@7
```

> ⚠️ Do **not** install `hard-source-webpack-plugin` — it's incompatible with webpack 5. Use the built-in **Persistent Cache** instead (see Phase 4).

### 1.3 React 18 + Babel

```bash
npm install react@18 react-dom@18

npm install --save-dev \
  @babel/core@7 \
  @babel/preset-env@7 \
  @babel/preset-react@7 \
  @babel/preset-typescript@7 \  # if using TS
  @babel/plugin-transform-runtime@7 \
  @babel/runtime
```

---

## Phase 2 — Webpack Config Rewrite

> For full annotated config examples, see `references/config-examples.md`.

### 2.1 Mandatory Breaking Changes

| webpack 3 pattern | webpack 5 replacement |
|---|---|
| `CommonsChunkPlugin` | `optimization.splitChunks` |
| `ExtractTextPlugin` | `MiniCssExtractPlugin` |
| `UglifyJsPlugin` | `TerserPlugin` (built-in) |
| `optimize-css-assets-webpack-plugin` | `CssMinimizerPlugin` |
| `loaders: []` (v1/v2 compat) | `rules: []` (required) |
| `require.ensure()` | `import()` dynamic imports |
| `module.loaders` | `module.rules` |
| `webpack.optimize.OccurrenceOrderPlugin` | removed, built-in |
| `webpack.optimize.DedupePlugin` | removed, built-in |
| `new webpack.NamedModulesPlugin()` | `optimization.moduleIds: 'named'` |
| `new webpack.HashedModuleIdsPlugin()` | `optimization.moduleIds: 'deterministic'` |
| `output.filename` hash `[hash]` | use `[contenthash]` for long-term caching |
| Node.js polyfills (Buffer, process…) | **not auto-injected** — must opt-in (see §2.2) |

### 2.2 Node Polyfills (critical for React 18 projects)

webpack 5 removed automatic Node.js core polyfills. If your code or dependencies use `Buffer`, `process`, `crypto`, `stream`, `path`, etc., you must explicitly add them:

```bash
npm install --save-dev \
  buffer \
  process \
  stream-browserify \
  crypto-browserify \
  path-browserify
```

Then in `webpack.config.js`:

```js
const { ProvidePlugin } = require('webpack');

module.exports = {
  resolve: {
    fallback: {
      buffer: require.resolve('buffer/'),
      process: require.resolve('process/browser'),
      stream: require.resolve('stream-browserify'),
      crypto: require.resolve('crypto-browserify'),
      path: require.resolve('path-browserify'),
    },
  },
  plugins: [
    new ProvidePlugin({
      Buffer: ['buffer', 'Buffer'],
      process: 'process/browser',
    }),
  ],
};
```

> **Best practice**: Only polyfill what you actually use. Run the build first without polyfills; the errors will tell you exactly which modules need fallbacks.

### 2.3 Output configuration

```js
output: {
  path: path.resolve(__dirname, 'dist'),
  filename: '[name].[contenthash:8].js',    // ← contenthash not hash
  chunkFilename: '[name].[contenthash:8].chunk.js',
  assetModuleFilename: 'assets/[hash][ext][query]',
  clean: true,   // ← replaces CleanWebpackPlugin
  publicPath: 'auto',  // or your CDN URL
},
```

### 2.4 Asset handling (replaces file-loader / url-loader)

webpack 5 has built-in **Asset Modules**. Remove `file-loader` and `url-loader`:

```js
module: {
  rules: [
    // Images
    {
      test: /\.(png|jpe?g|gif|svg|webp|avif)$/i,
      type: 'asset',
      parser: {
        dataUrlCondition: { maxSize: 8 * 1024 }, // inline if < 8kb
      },
    },
    // Fonts
    {
      test: /\.(woff2?|eot|ttf|otf)$/i,
      type: 'asset/resource',
    },
  ],
},
```

### 2.5 JavaScript / Babel rule

```js
{
  test: /\.[jt]sx?$/,
  exclude: /node_modules/,
  use: {
    loader: 'babel-loader',
    options: {
      cacheDirectory: true,   // ← critical for build speed
      presets: [
        ['@babel/preset-env', { targets: '> 0.5%, last 2 versions, not dead' }],
        ['@babel/preset-react', { runtime: 'automatic' }],  // ← no need to import React
      ],
    },
  },
},
```

> `runtime: 'automatic'` enables the new JSX transform for React 17+. Remove all `import React from 'react'` from files that only use JSX (optional but cleaner).

### 2.6 CSS / Sass rules

```js
// Production: extract to files
{
  test: /\.s?css$/,
  use: [
    isProd ? MiniCssExtractPlugin.loader : 'style-loader',
    {
      loader: 'css-loader',
      options: { modules: { auto: true }, sourceMap: !isProd },
    },
    'postcss-loader',
    'sass-loader',
  ],
},
```

---

## Phase 3 — Optimization Block

### 3.1 Code splitting (replaces CommonsChunkPlugin)

```js
optimization: {
  splitChunks: {
    chunks: 'all',
    cacheGroups: {
      vendor: {
        test: /[\\/]node_modules[\\/]/,
        name: 'vendors',
        chunks: 'all',
        priority: 10,
      },
      react: {
        test: /[\\/]node_modules[\\/](react|react-dom|react-router)[\\/]/,
        name: 'react-vendor',
        chunks: 'all',
        priority: 20,
      },
    },
  },
  runtimeChunk: 'single',   // ← stable long-term caching
},
```

### 3.2 Minimizers

```js
optimization: {
  minimize: true,
  minimizer: [
    new TerserPlugin({
      parallel: true,                      // ← uses all CPU cores
      terserOptions: { compress: { drop_console: true } },
    }),
    new CssMinimizerPlugin(),
  ],
},
```

---

## Phase 4 — Performance & Build Speed

> Full tuning details → `references/performance.md`

### Quick wins (implement in order of impact)

1. **Persistent filesystem cache** (biggest win, often 5–10× faster rebuilds)
   ```js
   cache: {
     type: 'filesystem',
     buildDependencies: { config: [__filename] },
   },
   ```

2. **babel-loader cacheDirectory** — already covered in §2.5 above.

3. **Restrict loader scope** — always use `include` or tight `exclude`:
   ```js
   { test: /\.jsx?$/, include: path.resolve(__dirname, 'src'), use: 'babel-loader' }
   ```

4. **thread-loader** for heavy JS transforms:
   ```bash
   npm install --save-dev thread-loader
   ```
   ```js
   use: ['thread-loader', 'babel-loader']
   ```

5. **resolve.extensions** — keep the list short and ordered by frequency:
   ```js
   resolve: { extensions: ['.tsx', '.ts', '.jsx', '.js'] }
   ```

6. **Production-only source maps** — use `eval-cheap-module-source-map` for dev, `source-map` for prod only.

7. **Target modern browsers** — set `target: ['web', 'es2017']` to skip unnecessary transpilation.

---

## Phase 5 — webpack-dev-server v5 Changes

`webpack-dev-server` v5 has significant API changes from v3:

```js
// Old v3 devServer options → New v5 equivalents
{
  devServer: {
    // OLD: contentBase: './dist'
    static: { directory: path.join(__dirname, 'dist') },

    // OLD: hot: true (worked automatically)
    hot: true,   // still works, but HMR is improved

    // OLD: historyApiFallback: true
    historyApiFallback: true,   // still the same

    // OLD: proxy: { '/api': 'http://localhost:3000' }
    proxy: [{ context: ['/api'], target: 'http://localhost:3000' }],  // ← array now

    // OLD: disableHostCheck: true  (security risk — remove entirely)
    // Use allowedHosts: 'all' only in dev if needed
    allowedHosts: 'auto',

    port: 3000,
    open: true,
  },
}
```

---

## Phase 6 — React 18 Specific Steps

1. **Update entry point** — React 18 uses `createRoot`:
   ```jsx
   // OLD
   import ReactDOM from 'react-dom';
   ReactDOM.render(<App />, document.getElementById('root'));

   // NEW (React 18)
   import { createRoot } from 'react-dom/client';
   const root = createRoot(document.getElementById('root'));
   root.render(<App />);
   ```

2. **Concurrent features** — If using Suspense + lazy, no webpack config changes needed; webpack 5's dynamic `import()` natively supports React.lazy.

3. **React Fast Refresh** (replaces react-hot-loader):
   ```bash
   npm install --save-dev @pmmmwh/react-refresh-webpack-plugin react-refresh
   ```
   ```js
   // webpack.config.js (dev only)
   const ReactRefreshPlugin = require('@pmmmwh/react-refresh-webpack-plugin');

   plugins: [new ReactRefreshPlugin()],

   // In babel-loader options:
   plugins: [require.resolve('react-refresh/babel')],
   ```

---

## Phase 7 — Validation Checklist

Run through this before merging:

- [ ] `npm run build` completes without errors
- [ ] `npm run build` output size is ≤ previous bundle (use `webpack-bundle-analyzer`)
- [ ] `npm start` (dev server) starts and HMR works
- [ ] Source maps open correctly in browser DevTools
- [ ] CSS is extracted to separate file(s) in production
- [ ] Images and fonts load correctly
- [ ] Code splitting: check Network tab for chunk loading
- [ ] Long-term caching: `[contenthash]` in filenames
- [ ] Environment variables (`process.env.NODE_ENV`) resolve correctly
- [ ] No `console.*` in production bundle (if using Terser drop_console)
- [ ] React DevTools shows "Concurrent Mode" indicator (React 18)
- [ ] Run `npx webpack-bundle-analyzer dist/stats.json` to inspect output

---

## References

Load these on demand when the user needs deeper detail:

| File | When to load |
|---|---|
| `references/config-examples.md` | User needs a full annotated webpack.config.js template |
| `references/errors-and-fixes.md` | User hits a build error or runtime breakage |
| `references/performance.md` | User wants to squeeze out maximum build speed |
