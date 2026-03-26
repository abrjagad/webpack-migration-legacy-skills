# Config Examples Reference

Full, annotated `webpack.config.js` templates for a React 18 project migrated from webpack 3.

---

## Table of Contents
1. [webpack.common.js](#common)
2. [webpack.dev.js](#dev)
3. [webpack.prod.js](#prod)
4. [babel.config.js](#babel)
5. [postcss.config.js](#postcss)
6. [package.json scripts](#scripts)

---

## 1. webpack.common.js <a name="common"></a>

Shared config merged into both dev and prod.

```js
// webpack.common.js
const path = require('path');
const HtmlWebpackPlugin = require('html-webpack-plugin');
const { DefinePlugin } = require('webpack');

const SRC = path.resolve(__dirname, 'src');
const DIST = path.resolve(__dirname, 'dist');

module.exports = {
  // ── Entry ────────────────────────────────────────────────────────────────
  entry: {
    main: path.join(SRC, 'index.jsx'),
  },

  // ── Output ───────────────────────────────────────────────────────────────
  output: {
    path: DIST,
    filename: '[name].[contenthash:8].js',
    chunkFilename: '[name].[contenthash:8].chunk.js',
    assetModuleFilename: 'assets/[hash:8][ext][query]',
    publicPath: '/',
    clean: true,  // replaces CleanWebpackPlugin
  },

  // ── Target ───────────────────────────────────────────────────────────────
  // Tells webpack + babel what browsers to target — skip old transpilations
  target: ['web', 'es2017'],

  // ── Resolve ──────────────────────────────────────────────────────────────
  resolve: {
    extensions: ['.tsx', '.ts', '.jsx', '.js', '.json'],
    alias: {
      '@': SRC,   // import Foo from '@/components/Foo'
    },
    // Node core polyfills — only add what you actually need:
    fallback: {
      buffer: false,    // change to require.resolve('buffer/') if needed
      process: false,   // change to require.resolve('process/browser') if needed
      stream: false,
      crypto: false,
      path: false,
      fs: false,
    },
  },

  // ── Module Rules ─────────────────────────────────────────────────────────
  module: {
    rules: [
      // JavaScript / TypeScript / JSX / TSX
      {
        test: /\.[jt]sx?$/,
        include: SRC,   // ← always scope loaders to src, never whole project
        use: {
          loader: 'babel-loader',
          options: {
            cacheDirectory: true,          // filesystem cache for babel transforms
            cacheCompression: false,       // don't gzip cache, faster reads
          },
        },
      },

      // Images — webpack 5 Asset Modules (replaces file-loader + url-loader)
      {
        test: /\.(png|jpe?g|gif|webp|avif|ico)$/i,
        type: 'asset',
        parser: {
          dataUrlCondition: { maxSize: 8 * 1024 }, // inline if < 8 KB
        },
      },

      // SVG — inline as React component (requires @svgr/webpack)
      {
        test: /\.svg$/i,
        issuer: /\.[jt]sx?$/,
        use: [
          { loader: '@svgr/webpack', options: { exportType: 'named' } },
          'url-loader',
        ],
      },

      // Fonts
      {
        test: /\.(woff2?|eot|ttf|otf)$/i,
        type: 'asset/resource',
        generator: { filename: 'fonts/[hash:8][ext]' },
      },
    ],
  },

  // ── Plugins ──────────────────────────────────────────────────────────────
  plugins: [
    new HtmlWebpackPlugin({
      template: path.join(SRC, 'index.html'),
      inject: true,
    }),

    new DefinePlugin({
      'process.env.NODE_ENV': JSON.stringify(process.env.NODE_ENV || 'development'),
      // Add your own env vars here:
      // 'process.env.API_URL': JSON.stringify(process.env.API_URL),
    }),
  ],

  // ── Optimization (shared) ────────────────────────────────────────────────
  optimization: {
    runtimeChunk: 'single',   // shared runtime chunk for stable hashing
    splitChunks: {
      chunks: 'all',
      maxInitialRequests: 30,
      maxAsyncRequests: 30,
      cacheGroups: {
        // React ecosystem — high-priority, separate chunk
        reactVendor: {
          test: /[\\/]node_modules[\\/](react|react-dom|react-router[-\w]*)[\\/]/,
          name: 'vendor-react',
          chunks: 'all',
          priority: 30,
          enforce: true,
        },
        // All other node_modules
        vendors: {
          test: /[\\/]node_modules[\\/]/,
          name: 'vendor',
          chunks: 'all',
          priority: 10,
        },
      },
    },
  },
};
```

---

## 2. webpack.dev.js <a name="dev"></a>

Development-specific config merged on top of common.

```js
// webpack.dev.js
const { merge } = require('webpack-merge');
const ReactRefreshPlugin = require('@pmmmwh/react-refresh-webpack-plugin');
const common = require('./webpack.common');

module.exports = merge(common, {
  mode: 'development',

  // Fast, accurate source maps for dev
  devtool: 'eval-cheap-module-source-map',

  // ── Persistent Cache ─────────────────────────────────────────────────────
  // Stores module graph on disk — subsequent cold starts are 5-10× faster
  cache: {
    type: 'filesystem',
    buildDependencies: {
      config: [__filename],   // invalidate cache if this config changes
    },
    name: 'development-cache',
  },

  // ── Dev Server ───────────────────────────────────────────────────────────
  devServer: {
    static: {
      directory: require('path').join(__dirname, 'public'),
    },
    hot: true,
    port: 3000,
    open: true,
    compress: true,
    historyApiFallback: true,   // for React Router
    proxy: [
      // Example: proxy /api calls to a local backend
      // { context: ['/api'], target: 'http://localhost:8080' },
    ],
  },

  // ── CSS (inject into DOM via style-loader) ────────────────────────────────
  module: {
    rules: [
      {
        test: /\.s?css$/,
        use: [
          'style-loader',
          {
            loader: 'css-loader',
            options: {
              modules: { auto: true, localIdentName: '[name]__[local]' },
              sourceMap: true,
            },
          },
          'postcss-loader',
          {
            loader: 'sass-loader',
            options: { sourceMap: true },
          },
        ],
      },
    ],
  },

  plugins: [
    new ReactRefreshPlugin(),   // React Fast Refresh HMR
  ],
});
```

---

## 3. webpack.prod.js <a name="prod"></a>

Production config — minification, extraction, hashing.

```js
// webpack.prod.js
const { merge } = require('webpack-merge');
const MiniCssExtractPlugin = require('mini-css-extract-plugin');
const CssMinimizerPlugin = require('css-minimizer-webpack-plugin');
const TerserPlugin = require('terser-webpack-plugin');
const common = require('./webpack.common');

module.exports = merge(common, {
  mode: 'production',

  // Full source maps for production error tracking (remove if not needed)
  devtool: 'source-map',

  // ── Persistent Cache ─────────────────────────────────────────────────────
  cache: {
    type: 'filesystem',
    buildDependencies: { config: [__filename] },
    name: 'production-cache',
  },

  // ── CSS: extract to files ─────────────────────────────────────────────────
  module: {
    rules: [
      {
        test: /\.s?css$/,
        use: [
          MiniCssExtractPlugin.loader,
          {
            loader: 'css-loader',
            options: {
              modules: { auto: true, localIdentName: '[hash:base64:8]' },
              sourceMap: true,
            },
          },
          'postcss-loader',
          'sass-loader',
        ],
      },
    ],
  },

  plugins: [
    new MiniCssExtractPlugin({
      filename: 'css/[name].[contenthash:8].css',
      chunkFilename: 'css/[name].[contenthash:8].chunk.css',
    }),
  ],

  // ── Minimizers ────────────────────────────────────────────────────────────
  optimization: {
    minimize: true,
    minimizer: [
      new TerserPlugin({
        parallel: true,   // uses all CPU cores — big speed boost on CI
        terserOptions: {
          compress: {
            drop_console: true,       // strip console.* in production
            drop_debugger: true,
            pure_funcs: ['console.log'],
          },
          format: { comments: false },
        },
        extractComments: false,
      }),
      new CssMinimizerPlugin({
        parallel: true,
      }),
    ],
  },

  performance: {
    hints: 'warning',
    maxEntrypointSize: 512 * 1024,   // 512 KB warning threshold
    maxAssetSize: 512 * 1024,
  },
});
```

---

## 4. babel.config.js <a name="babel"></a>

```js
// babel.config.js
module.exports = (api) => {
  const isTest = api.env('test');

  return {
    presets: [
      [
        '@babel/preset-env',
        {
          // Let browserslist (or webpack's target) drive what gets transpiled
          useBuiltIns: 'usage',
          corejs: 3,
          // In test env, compile to CommonJS for Jest
          modules: isTest ? 'commonjs' : false,
        },
      ],
      [
        '@babel/preset-react',
        {
          runtime: 'automatic',   // ← enables new JSX transform (no React import needed)
          development: process.env.NODE_ENV !== 'production',
        },
      ],
      // Uncomment if using TypeScript:
      // ['@babel/preset-typescript', { allExtensions: true, isTSX: true }],
    ],

    plugins: [
      '@babel/plugin-transform-runtime',
      // React Fast Refresh in dev only:
      ...(process.env.NODE_ENV === 'development'
        ? ['react-refresh/babel']
        : []),
    ],
  };
};
```

---

## 5. postcss.config.js <a name="postcss"></a>

```js
// postcss.config.js
module.exports = {
  plugins: [
    require('autoprefixer'),
    // Uncomment for modern CSS:
    // require('postcss-preset-env')({ stage: 3 }),
  ],
};
```

```bash
npm install --save-dev postcss autoprefixer
```

---

## 6. package.json scripts <a name="scripts"></a>

```json
{
  "scripts": {
    "start":        "webpack serve --config webpack.dev.js",
    "build":        "NODE_ENV=production webpack --config webpack.prod.js",
    "build:analyze":"webpack --config webpack.prod.js --env analyze",
    "build:stats":  "webpack --config webpack.prod.js --json > dist/stats.json",
    "analyze":      "webpack-bundle-analyzer dist/stats.json"
  }
}
```

```bash
npm install --save-dev webpack-bundle-analyzer
```
