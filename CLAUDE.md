# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

Mempool (mempool.space) is a Bitcoin mempool visualizer, blockchain explorer, and API service. The repository is a monorepo with three interdependent components:

- **`backend/`** — TypeScript / Node.js API server (Express + WebSocket). Ingests data from Bitcoin Core, an Electrum/Esplora server, and MariaDB; serves the REST/WS API.
- **`frontend/`** — Angular 20 single-page app (SSR-capable). The *same* codebase builds both **mempool.space** and **liquid.network**, selected at config time.
- **`rust/gbt/`** — Rust re-implementation of Bitcoin's `getBlockTemplate` block-assembly algorithm, compiled to a native Node addon via [napi-rs](https://napi.rs) and consumed by the backend.

`production/`, `docker/`, and the top-level `nginx*.conf` files are deployment tooling; `contributors/` holds CLA sign-off files.

## Toolchain

- **Node `v24.13.0`** (see `.nvmrc`); CI uses **npm `11.12.0`**. (The component READMEs still say Node 20.x — follow `.nvmrc`/CI.)
- **Rust** is required to build the backend (the `rust/gbt` addon). The toolchain version is pinned in `rust/gbt/rust-toolchain`.
- `backend/` and `frontend/` have **separate `package.json` / `node_modules`** — `npm install` and run scripts from within each directory, not the repo root.

## Backend (`cd backend`)

```bash
npm install              # also runs `preinstall`, which builds rust/gbt and copies it into backend/rust-gbt/
npm run build            # tsc -> dist/, then copies runtime resources
npm run start            # node dist/index.js  (needs mempool-config.json)
npm run lint             # eslint . --ext .ts   (lint:fix to autofix)
npm run test             # jest unit tests with coverage
npm run test:ci          # CI=true jest
```

- **Config:** copy `mempool-config.sample.json` → `mempool-config.json`. Override the path with `MEMPOOL_CONFIG_FILE=/path/to/config.json`. All config is centralized and typed in `src/config.ts` (file values are merged over defaults).
- **Dev watcher:** `nodemon src/index.ts --ignore cache/` (avoids the manual recompile/restart cycle).
- **Runtime CLI flags** (passed via npm, e.g. `npm run start --update-pools`): `--update-pools`, `--reindex-blocks`, `--reindex=blocks,hashrates` (truncates the listed tables, then re-indexes — 5s grace period before truncation).

### Running a single backend test

```bash
npx jest src/__tests__/api/difficulty-adjustment.test.ts   # one file
npx jest -t "name of test"                                  # by test name
```

### Backend integration tests (require MariaDB)

Integration tests live in `src/__integration_tests__/`, use `jest.integration.config.ts`, and run **sequentially** (`maxWorkers: 1`).

```bash
npm run test:with-db        # spins up MariaDB via docker-compose.test.yml, builds, runs, tears down
npm run test:integration    # runs against an already-running DB (uses mempool-config.test.json)
```

## Frontend (`cd frontend`)

```bash
npm install
npm run config:defaults:mempool   # configure as mempool.space  (or config:defaults:liquid for liquid.network)
npm run serve:local-prod          # dev server on :4200, proxying the API to production mempool.space (no local backend needed)
npm run serve                     # dev server proxying to a LOCAL backend
npm run build                     # production build with i18n (--localize)
npm run lint                      # eslint (flat config: eslint.config.js)
```

- **Config is generated, not hand-edited at runtime.** `generate-config.js` reads `mempool-frontend-config.json` (sample: `mempool-frontend-config.sample.json`) to produce `src/resources/config.js`; `update-config.js KEY=value …` flips individual keys; `generate-themes.js` builds the theme CSS. The `serve`/`build` npm scripts run these automatically — running `ng serve`/`ng build` directly will skip them.
- **`BASE_MODULE`** (`mempool` vs `liquid`) is what makes one codebase serve two sites; it selects the `src/index.<module>.html` entry and feature flags. Enterprise/white-label variants use `custom-*-config.json` + `index.mempool.<name>.html`.
- **Serve "configurations"** (`local`, `local-prod`, `parameterized`, `mixed`, `local-esplora`) are Angular CLI configurations in `angular.json`, each paired with a `proxy.conf.*.js` that decides where API/WS calls are proxied.

### Frontend tests (Cypress E2E)

Angular/Karma unit tests (`npm run test`) exist but are **not run in CI**. The real coverage is Cypress E2E, split by module (`mempool`, `liquid`, `testnet4`):

```bash
npm run config:defaults:mempool && npm run cypress:open   # interactive
npm run config:defaults:mempool && npm run cypress:run    # headless
```

## Rust gbt (`cd rust/gbt`)

You rarely build this directly — `backend`'s `preinstall` hook does it. When iterating on the Rust code:

```bash
npm run build-release   # napi build --release; produces gbt.<target-triple>.node
npm run to-backend      # copies the built addon + glue into backend/rust-gbt/
npm test                # cargo test
```

The backend imports it as the `rust-gbt` dependency (`file:./rust-gbt`) and its use is gated by `MEMPOOL.RUST_GBT` in the config.

## Backend architecture

- **Entry point `src/index.ts`** — a `Server` class wiring up Express routes, the WebSocket server, the disk/redis caches, and the periodic mempool/block poll loop. Optionally forks worker processes when `MEMPOOL.SPAWN_CLUSTER_PROCS > 0`.
- **Data-source abstraction:** `src/api/bitcoin/` uses an abstract factory (`bitcoin-api-factory.ts` → `bitcoin-api-abstract-factory.ts`) to select the backing source based on `config.MEMPOOL.BACKEND`: `esplora`, `electrum`, or `none`. Block/tx/address lookups go through this interface, with Bitcoin Core RPC (`bitcoin-client.ts`) used directly for node-level calls.
- **`src/api/`** holds the domain logic: `mempool.ts`, `blocks.ts`, `mempool-blocks.ts`, `websocket-handler.ts`, plus feature areas `mining/`, `lightning/`, `liquid/`, `prices/`, `acceleration/`, `statistics/`, and `services/`.
- **`src/repositories/`** — all MariaDB access (Blocks, Hashrates, Pools, Prices, CPFP, …). `src/database.ts` owns the connection pool; `src/api/database-migration.ts` applies schema migrations on startup (bump its version when changing schema).
- **`src/tasks/`** — background updaters (`pools-updater.ts`, `price-updater.ts`, `lightning/` sync services). **`src/indexer.ts`** drives historical indexing of blocks/stats.
- **Supported networks** (`config.MEMPOOL.NETWORK`): `mainnet`, `testnet`, `testnet4`, `signet`, `liquid`, `liquidtestnet`, `regtest`.

## Frontend architecture

- Angular standalone-ish modules under `src/app/`: `components/` (~110 explorer/visualization components), feature modules `lightning/`, `liquid/`, `dashboard/`, `graphs/`, `docs/`, and `shared/`.
- **`services/`** is the data layer: `api.service.ts` / `electrs-api.service.ts` (REST), `websocket.service.ts` (live updates), `state.service.ts` (central app state + the active network/module), plus `cache.service.ts`, `price.service.ts`, etc.
- Graphs use **ECharts** (`ngx-echarts`); UI uses **ng-bootstrap**. SSR is served via `server.ts` / `server.run.ts`.
- **i18n:** strings are localized into 20+ locales via Transifex; `npm run i18n-extract-from-source` regenerates `src/locale/messages.xlf`, and production builds localize with `--localize`.

## Conventions & gotchas

- **Backend async-safety lint rule.** A custom ESLint rule `local-rules/no-unhandled-await` (error) plus `@typescript-eslint/no-floating-promises` (error) forbid unhandled `await`/`void` of a promise inside an `@asyncSafe` context. To satisfy it, do one of: wrap in `try/catch`, use `.catch(...)` / `.then(onFulfilled, onRejected)` / `Promise.allSettled`, annotate the *callee* with a `@asyncSafe` JSDoc tag, or mark the surrounding function/block `@asyncUnsafe`. This is the most common lint failure when adding backend async code. (Integration tests opt out of both rules.)
- **ESLint versions differ:** backend is ESLint 8 with legacy `.eslintrc.js`; frontend is ESLint 9 with flat `eslint.config.js`. Don't copy lint config between them.
- **CI** (`.github/workflows/ci.yml`) runs backend (lint + unit tests + build), frontend (lint + build), and Cypress E2E, each in `dev` and `prod` dependency flavors. Backend DB integration tests run in `backend-integration.yml`. Match these locally before pushing.
- **Contributing:** first-time contributors add `contributors/<github_username>.txt` agreeing to the CLA, and commits should be GPG-signed (`git config commit.gpgsign true`).
</content>
