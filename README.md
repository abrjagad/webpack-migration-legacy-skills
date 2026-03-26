# webpack-migration-legacy-skills

A Claude Code skill that guides webpack 3 → webpack 5 migrations for React 18 projects.

## What this skill does

When loaded into Claude Code, this skill gives Claude deep, structured knowledge to assist with migrating legacy webpack 3 configurations to webpack 5 — covering dependency upgrades, config rewrites, React 18 compatibility, performance tuning, and common breaking-change fixes.

## Skill trigger

Claude will automatically use this skill when you mention:

- "webpack 3 to 5", "webpack upgrade", "webpack migration", "modernize build"
- Showing a `webpack.config.js` with v3-era patterns (`ExtractTextPlugin`, `CommonsChunkPlugin`, `UglifyJsPlugin`, `loaders: []`, etc.)
- "optimize webpack build times"

## What's covered

| Phase | Topic |
|---|---|
| 0 | Pre-migration audit checklist |
| 1 | Node.js + core dependency upgrades |
| 2 | webpack config rewrite (breaking changes, asset modules, CSS, Babel) |
| 3 | Optimization block (code splitting, minimizers) |
| 4 | Build performance (persistent cache, thread-loader, tree shaking) |
| 5 | webpack-dev-server v5 API changes |
| 6 | React 18 specific steps (createRoot, Fast Refresh) |
| 7 | Validation checklist |

## Repository structure

```
skills/
├── SKILL.md                        # Main skill — full migration workflow
└── references/
    ├── config-examples.md          # Annotated webpack.common/dev/prod.js templates
    ├── errors-and-fixes.md         # 15 common migration errors with fixes
    └── performance.md              # Build speed tuning (cache, parallelization, etc.)
```

## Reference files

The skill loads these on demand:

- **`config-examples.md`** — complete annotated `webpack.common.js`, `webpack.dev.js`, `webpack.prod.js`, `babel.config.js`, `postcss.config.js`, and `package.json` scripts
- **`errors-and-fixes.md`** — fixes for errors like `CommonsChunkPlugin is not a constructor`, `Can't resolve 'crypto'`, `ReactDOM.render is no longer supported`, broken HMR, CSS Modules class name changes, and more
- **`performance.md`** — detailed tuning for persistent filesystem cache, `thread-loader`, loader scoping, tree shaking, lazy compilation, bundle analysis, and CI/CD caching

## Key migration changes covered

| webpack 3 | webpack 5 |
|---|---|
| `CommonsChunkPlugin` | `optimization.splitChunks` |
| `ExtractTextPlugin` | `MiniCssExtractPlugin` |
| `UglifyJsPlugin` | `TerserPlugin` (built-in) |
| `file-loader` / `url-loader` | Asset Modules (built-in) |
| `hard-source-webpack-plugin` | Persistent filesystem cache (built-in) |
| `react-hot-loader` | React Fast Refresh |
| `ReactDOM.render` | `createRoot` (React 18) |
| `[hash]` in filenames | `[contenthash]` |
| Auto Node.js polyfills | Explicit `resolve.fallback` |

## Usage

Install this skill into your Claude Code setup by pointing Claude at the `skills/` directory, or copy `SKILL.md` and the `references/` folder into your project's skills location.
