# Performance Reference

Detailed build performance tuning for webpack 5 + React 18 projects.
Organized from highest to lowest impact.

---

## Table of Contents
1. [Persistent Filesystem Cache](#cache)
2. [Parallelization](#parallel)
3. [Loader Scoping](#scope)
4. [Module Resolution Tuning](#resolve)
5. [Source Map Strategy](#sourcemaps)
6. [Tree Shaking](#tree-shaking)
7. [Lazy Compilation (dev only)](#lazy)
8. [Bundle Analysis](#analysis)
9. [CI / CD Build Speed](#ci)
10. [Benchmarking Before & After](#benchmark)

---

## 1. Persistent Filesystem Cache <a name="cache"></a>

**Impact: ⭐⭐⭐⭐⭐ — typically 5–10× faster cold restarts after first build**

webpack 5's built-in filesystem cache stores the module graph, resolved dependencies, and compiled modules on disk between builds.

```js
cache: {
  type: 'filesystem',

  // Invalidate if ANY of these files change:
  buildDependencies: {
    config: [
      require.resolve('./webpack.common.js'),
      require.resolve('./webpack.dev.js'),   // or webpack.prod.js
      require.resolve('./babel.config.js'),
      require.resolve('./package.json'),
    ],
  },

  // Separate caches per environment — prevents dev cache poisoning prod:
  name: `${process.env.NODE_ENV || 'development'}-cache`,

  // Default location: node_modules/.cache/webpack
  // Override if needed:
  // cacheDirectory: path.resolve(__dirname, '.webpack-cache'),

  // Max age for entries (default: 5184000 = 60 days)
  maxAge: 172800000,  // 2 days — good for active development
},
```

**Gotchas:**
- Add `node_modules/.cache` to `.gitignore`.
- On CI, cache the `.cache/webpack` directory between runs (see §CI section).
- If you see stale output, delete `.cache/webpack` and rebuild.

---

## 2. Parallelization <a name="parallel"></a>

### 2.1 thread-loader (heavy transforms)

Offloads expensive loaders to a worker pool. Most effective for large TypeScript or Babel transforms.

```bash
npm install --save-dev thread-loader
```

```js
{
  test: /\.[jt]sx?$/,
  include: path.resolve(__dirname, 'src'),
  use: [
    {
      loader: 'thread-loader',
      options: {
        workers: require('os').cpus().length - 1,
        poolTimeout: Infinity,  // keep workers alive between builds in watch mode
      },
    },
    {
      loader: 'babel-loader',
      options: { cacheDirectory: true },
    },
  ],
},
```

**When NOT to use thread-loader:**
- On projects with < 200 JS/TS files — worker spawn overhead may be larger than the gain.
- With `ts-loader` using `transpileOnly: false` (type checking is incompatible with thread isolation).

### 2.2 TerserPlugin parallel (production)

Already parallel by default in webpack 5, but make it explicit:
```js
new TerserPlugin({ parallel: true })
```

### 2.3 CssMinimizerPlugin parallel

```js
new CssMinimizerPlugin({ parallel: true })
```

---

## 3. Loader Scoping <a name="scope"></a>

**Impact: ⭐⭐⭐⭐ — prevents accidentally running heavy transforms on node_modules**

Always use `include` to scope loaders to your source files:

```js
// ✗ Bad — transforms entire project including node_modules
{ test: /\.jsx?$/, use: 'babel-loader' }

// ✓ Good — only transforms your source
{ test: /\.jsx?$/, include: path.resolve(__dirname, 'src'), use: 'babel-loader' }

// ✓ Also acceptable:
{ test: /\.jsx?$/, exclude: /node_modules/, use: 'babel-loader' }
```

If you need to transform specific node_modules (e.g., they ship untranspiled ESM):
```js
{
  test: /\.jsx?$/,
  exclude: /node_modules\/(?!(my-esm-only-package|another-package)\/).*/,
  use: 'babel-loader',
}
```

---

## 4. Module Resolution Tuning <a name="resolve"></a>

### 4.1 Keep extensions list minimal and ordered by frequency

```js
resolve: {
  // Order matters — webpack tries each in sequence
  extensions: ['.tsx', '.ts', '.jsx', '.js'],
  // Don't include: '.json', '.node' unless you import them constantly
}
```

### 4.2 Disable symlinks if you don't use them

```js
resolve: {
  symlinks: false,  // skip symlink resolution — saves stat() calls
}
```

### 4.3 Use resolve.alias for frequent deep imports

```js
resolve: {
  alias: {
    '@': path.resolve(__dirname, 'src'),
    '@components': path.resolve(__dirname, 'src/components'),
    '@hooks': path.resolve(__dirname, 'src/hooks'),
  },
}
```

### 4.4 Limit resolve.modules

```js
resolve: {
  // Default looks up the full directory tree — narrow it:
  modules: [path.resolve(__dirname, 'src'), 'node_modules'],
}
```

---

## 5. Source Map Strategy <a name="sourcemaps"></a>

Source maps have a large impact on build time. Choose carefully:

| Environment | Recommended `devtool` | Build Speed | Quality |
|---|---|---|---|
| Dev (watch) | `eval-cheap-module-source-map` | ⭐⭐⭐⭐ | ✓ original lines |
| Dev (one-off) | `eval-source-map` | ⭐⭐⭐ | ✓✓ lines + columns |
| Production | `source-map` | ⭐⭐ | ✓✓✓ full |
| Production (no tracking) | `false` | ⭐⭐⭐⭐⭐ | none |

**Best practice:** Use `eval-cheap-module-source-map` in dev for the best balance of rebuild speed and debuggability. Only generate full `source-map` in production if you have error tracking (Sentry, Datadog, etc.).

---

## 6. Tree Shaking <a name="tree-shaking"></a>

webpack 5 tree-shakes ESM modules by default in production mode. Ensure your project takes advantage:

### 6.1 Mark your package as side-effect free

```json
// package.json
{
  "sideEffects": [
    "*.css",
    "*.scss",
    "src/polyfills.js"
  ]
}
```
Setting `"sideEffects": false` allows webpack to remove unused exports aggressively.

### 6.2 Use named imports from libraries

```js
// ✗ Imports entire lodash bundle
import _ from 'lodash';

// ✓ Imports only the `debounce` function
import debounce from 'lodash/debounce';
// or with ESM version:
import { debounce } from 'lodash-es';
```

### 6.3 Ensure Babel doesn't compile away ESM

In `babel.config.js`, set `modules: false` for non-test environments:
```js
['@babel/preset-env', {
  modules: process.env.NODE_ENV === 'test' ? 'commonjs' : false,
}]
```

---

## 7. Lazy Compilation (dev only) <a name="lazy"></a>

For large apps, webpack 5 can compile entry points and dynamic imports on-demand:

```js
// webpack.dev.js
experiments: {
  lazyCompilation: {
    imports: true,   // lazy-compile dynamic import() chunks
    entries: false,  // don't lazy-compile entry points (breaks HMR)
  },
},
```

**When to use:** Apps with 50+ routes or heavy dynamic imports that make the initial dev server startup painfully slow.

---

## 8. Bundle Analysis <a name="analysis"></a>

### 8.1 webpack-bundle-analyzer

```bash
npm install --save-dev webpack-bundle-analyzer

# Generate stats
webpack --config webpack.prod.js --json > dist/stats.json

# Analyze
npx webpack-bundle-analyzer dist/stats.json
```

Or integrate into the config:
```js
// webpack.prod.js
const { BundleAnalyzerPlugin } = require('webpack-bundle-analyzer');

plugins: [
  ...(process.env.ANALYZE ? [new BundleAnalyzerPlugin()] : []),
],
```
```json
"build:analyze": "ANALYZE=true npm run build"
```

### 8.2 speed-measure-webpack-plugin

Measures time spent in each plugin and loader:

```bash
npm install --save-dev speed-measure-webpack-plugin
```

```js
const SpeedMeasurePlugin = require('speed-measure-webpack-plugin');
const smp = new SpeedMeasurePlugin();

module.exports = smp.wrap(require('./webpack.prod.js'));
```

---

## 9. CI / CD Build Speed <a name="ci"></a>

### 9.1 Cache webpack's persistent cache across CI runs

**GitHub Actions example:**
```yaml
- name: Cache webpack build
  uses: actions/cache@v3
  with:
    path: node_modules/.cache/webpack
    key: webpack-${{ runner.os }}-${{ hashFiles('**/package-lock.json', 'webpack.*.js', 'babel.config.js') }}
    restore-keys: |
      webpack-${{ runner.os }}-
```

### 9.2 Cache node_modules

```yaml
- name: Cache node_modules
  uses: actions/cache@v3
  with:
    path: node_modules
    key: node-${{ runner.os }}-${{ hashFiles('**/package-lock.json') }}
```

### 9.3 Set NODE_ENV in CI

Ensure CI sets `NODE_ENV=production` for production builds — this enables webpack's built-in optimizations and disables dev-only code.

### 9.4 Parallelize type checking

If using TypeScript with `ts-loader`, use `fork-ts-checker-webpack-plugin` to run type checking in a separate process, not blocking webpack's main thread:

```bash
npm install --save-dev fork-ts-checker-webpack-plugin
```

```js
const ForkTsCheckerPlugin = require('fork-ts-checker-webpack-plugin');

plugins: [new ForkTsCheckerPlugin()]
```

---

## 10. Benchmarking Before & After <a name="benchmark"></a>

Always measure before and after to confirm improvements:

```bash
# Measure full production build time
time npm run build

# Measure with webpack's built-in profiling
webpack --config webpack.prod.js --profile --json > dist/stats.json

# First build vs cached build comparison
rm -rf node_modules/.cache/webpack
time npm run build          # cold build time

time npm run build          # warm (cached) build time
```

**Typical improvements after full migration + optimization:**

| Metric | Before (webpack 3) | After (webpack 5 optimized) |
|---|---|---|
| Cold build | 60–120s | 20–40s |
| Cached rebuild | 40–80s | 3–8s |
| Dev HMR update | 3–8s | 0.2–1s |
| Bundle size | Baseline | -10 to -30% (tree shaking) |
