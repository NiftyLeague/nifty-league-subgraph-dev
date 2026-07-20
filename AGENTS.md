## Stack

- **Project:** The Graph subgraph (specVersion `1.3.0`), subgraph name `nifty-league-sepolia`
- **Purpose:** DEV/staging twin of `nifty-league-subgraph` — indexes `NiftyDegen` on Ethereum **Sepolia** (NOT mainnet)
- **Language:** **AssemblyScript** (compiled to WASM) via `@graphprotocol/graph-ts` `0.38.2` — _this is not standard TypeScript_; `src/*.ts` is AS, not Node TS
- **Toolchain:** `bun@1.3.14` (`packageManager` pinned), Node `24.18.0` — both pinned in `mise.toml`; CI installs via `jdx/mise-action`
- **CLI:** `@graphprotocol/graph-cli` `0.98.1`
- **TypeScript (host config only):** `typescript@^5.9.3` with `skipLibCheck` — used for editor/`tsc` type-checking, not runtime
- **ABI:** `./abis/NiftyDegen.json`
- **Lint:** `eslint@^9.39.5` + `typescript-eslint@^8.64.0` (zero-warnings policy)
- **Format:** `prettier@^3.6.2` — `semi: false`, `singleQuote: true`, `trailingComma: "es5"`, `tabWidth: 2`, `printWidth: 100`
- **AS integration tests:** `matchstick-as@0.6.0`
- **Deploy target:** Graph Studio (`https://api.studio.thegraph.com/deploy/`); local: `localhost:8020` (Graph node) + `localhost:5001` (IPFS)

## Commands

| Script                                     | Real command                    | Purpose                                                                                    |
| ------------------------------------------ | ------------------------------- | ------------------------------------------------------------------------------------------ |
| install                                    | `bun install --frozen-lockfile` | Install deps (CI exact)                                                                    |
| codegen                                    | `bun run codegen`               | `graph codegen` — emit AssemblyScript types from `schema.graphql` + ABIs into `generated/` |
| build                                      | `bun run build`                 | `graph build` — compile subgraph to WASM                                                   |
| test                                       | `bun run test`                  | `bun test bun-tests` — **Bun's native test runner** (NOT vitest, NOT jest)                 |
| test:graph                                 | `bun run test:graph`            | `graph test --version 0.6.0` — Matchstick/AssemblyScript integration tests (`tests/`)      |
| test:coverage                              | `bun run test:coverage`         | `node scripts/check-coverage.mjs` — runs tests with `--coverage`, enforces threshold       |
| lint                                       | `bun run lint`                  | `eslint . --max-warnings=0`                                                                |
| type-check                                 | `bun run type:check`            | `tsc --noEmit`                                                                             |
| format                                     | `bun run format`                | `prettier --write .`                                                                       |
| format:check                               | `bun run format:check`          | `prettier --check .`                                                                       |
| deploy                                     | `bun run deploy`                | `graph deploy --node https://api.studio.thegraph.com/deploy/ nifty-league-sepolia`         |
| dev / start                                | `bun run dev` / `bun run start` | Local Graph node + IPFS create+deploy                                                      |
| create-local / remove-local / deploy-local | `bun run create-local` etc.     | Local-node lifecycle against `localhost:8020` / `localhost:5001`                           |

Always: `bun install --frozen-lockfile` → `bun run codegen` → `bun run type:check` → `bun run lint` → `bun run test` → `bun run test:graph` → `bun run build`.

## Structure

```
nifty-league-subgraph-dev/
├── AGENTS.md
├── subgraph.yaml              # manifest — single data source, 8 event handlers
├── schema.graphql             # 10 entities: Contract, Character, Owner, TraitMap (domain) + Approval, ApprovalForAll, NameUpdated, OwnershipTransferred, Paused, Transfer, Unpaused (event, immutable)
├── networks.json              # Sepolia datasource: NiftyDegen @ 0x6adFF2BB4A465A885425e3bd4304A78BB659B12e, startBlock 5541380
├── abis/
│   └── NiftyDegen.json
├── src/                       # AssemblyScript mappings (WASM)
│   ├── nifty-degen.ts         # primary handler — imports `generated/NiftyDegen/NiftyDegen`, dispatches 8 events
│   ├── custom-mappings.ts     # domain logic: Character, Owner, Contract, TraitMap
│   ├── backgrounds.ts         # trait → background helpers
│   ├── background-classifier.ts
│   └── constants.ts
├── generated/                 # codegen output — NOT committed; produced by `bun run codegen`
├── tests/                     # Matchstick/AS integration tests
│   ├── nifty-degen.test.ts
│   └── nifty-degen-utils.ts
├── bun-tests/                 # Bun-native unit tests
│   └── background-classifier.test.ts
├── scripts/
│   └── check-coverage.mjs
├── eslint.config.mjs          # ESLint flat config
├── bunfig.toml
├── tsconfig.json              # TS 5.4.5 host config, skipLibCheck
├── mise.toml                  # pins node 24.18.0 + bun 1.3.14 (CI uses jdx/mise-action)
├── lint-staged.config.js      # prettier + eslint --fix on staged files
├── docker-compose.yml         # local Graph node + IPFS
├── bun.lock
└── package.json               # packageManager: bun@1.3.14, engines.node: 24.x
```

## Conventions & Gotchas

1. **Bun-only.** `packageManager: bun@1.3.14` pinned in `package.json`. `mise.toml` pins `node = "24.18.0"` + `bun = "1.3.14"`; CI installs via `jdx/mise-action`. **Never** `npm` / `yarn` / `npx`. All scripts via `bun run <script>`. Do not bump Bun or Node without updating `mise.toml` + lockfile.
2. **Test runner = `bun test` (Bun native).** `bun run test` invokes `bun test bun-tests` against `bun-tests/`. This is **not** vitest, **not** jest — do not import from `vitest` or `@jest/globals`. The `tests/` directory is a separate suite run via `bun run test:graph` (`graph test --version 0.6.0`, Matchstick/AS). The Solidity exception noted elsewhere does **not** apply here — there is no Solidity in this repo. `assemblyscript@0.19.23` and `wabt@1.0.39` are pinned.
3. **husky + lint-staged are active.** `lint-staged.config.js` runs `prettier --write` and `eslint --fix --max-warnings=0` on staged `*.{ts,tsx,js,json,md,mdx,yml,yaml,sol}`. **Never** `git commit --no-verify` — bypasses formatting and lint and will fail PR checks. Pre-commit hooks run automatically.
4. **This is the DEV twin — do not point at mainnet/prod.** The prod counterpart is `../nifty-league-subgraph` (mainnet, different address + startBlock). Before any contract change, verify `networks.json` and `subgraph.yaml` `datasource.address` + `datasource.startBlock` match the **Sepolia** deployment of `NiftyDegen` (`0x6adFF2BB4A465A885425e3bd4304A78BB659B12e`, startBlock `5541380`). Subgraph name is `nifty-league-sepolia`. `bun run deploy` targets Graph Studio (dev subgraph slug), not mainnet Studio.
5. **`generated/` is gitignored.** Run `bun run codegen` before `bun run build` or `bun run type:check` — `src/nifty-degen.ts` imports from `generated/NiftyDegen/NiftyDegen`, which does not exist until codegen runs.
6. **Zero-tolerance lint.** `eslint . --max-warnings=0` — any warning fails CI. Run `bun run lint` locally before pushing.
7. **Never run Hardhat compile here.** This repo has no Solidity sources, no `hardhat.config.*`, no `contracts/`. If you need to recompile `NiftyDegen` ABI, do it in `../nifty-smart-contracts` (or `../NiftyRoyaleFork`) and copy the updated JSON into `./abis/NiftyDegen.json`. Do not introduce Foundry/Hardhat toolchain into this repo.
8. **Prettier is non-default.** `semi: false`, `singleQuote: true`, `trailingComma: "es5"`. No trailing semicolons in `src/`. `bun run format` before `git commit` (lint-staged also runs it).
9. **Coverage gate.** `bun run test:coverage` parses Matchstick output for `Global test coverage:` / `Test coverage:` and exits non-zero below the threshold in `scripts/check-coverage.mjs` — do not silently lower it.
10. **Matchstick version pinned.** `graph test --version 0.6.0` is hard-coded in `package.json` to match `matchstick-as@0.6.0`; dropping the flag can pull a different binary.
11. **Local deploy assumes `docker-compose up`.** `bun run dev` / `deploy-local` will fail unless the local Graph node (`localhost:8020`) and IPFS (`localhost:5001`) from `docker-compose.yml` are running.
12. **No `README.md`.** Subgraph description lives in `subgraph.yaml`'s `description` field ("NiftyDegen Sepolia NFTs from NiftyLeague"). Do not add one unless asked.
13. **TypeScript host config ≠ runtime TS.** `tsc --noEmit` only validates the host-level TS (configs, scripts). Mapping correctness in `src/*.ts` is validated by `graph build` (WASM compile), not `tsc` — `graph-ts` types are stubs.
