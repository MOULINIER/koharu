@AGENTS.md

# CLAUDE.md

The imported `AGENTS.md` holds the binding project rules: no backward compatibility, ML port and upstream alignment, performance, and verification scope. This file adds working context. `packages/koharu/CLAUDE.md` adds rules for the Next.js frontend.

## Overview

Koharu is a desktop app for manga translation. It detects text, runs OCR, translates, inpaints, and typesets. It is a Rust 2024 Cargo workspace (`crates/`) with a Tauri v3 alpha shell running on CEF. The frontend is a Next.js/React app in a Bun workspace (`packages/`), and it draws the canvas through a Rust→WASM WebGPU module. ML inference goes through LibTorch, llama.cpp and stable-diffusion.cpp. The app loads all three dynamically at runtime; none is linked at build time.

## Commands

Prerequisites (from the README): Rust 1.97.1+, Bun 1.3.14+, LLVM 22.1.8+ (bindgen uses it).

```sh
bun install                       # also runs `cargo install tauri-cli --version =3.0.0-alpha.1` (postinstall)
bun dev                           # cargo tauri dev; builds the bridge WASM, runs Next on :3000
bun run build                     # cargo tauri build --no-bundle -> target/release

# Rust (these mirror CI; plain `cargo check` covers only default-member `koharu` and its deps)
cargo fmt -- --check
cargo check
cargo clippy -- -D warnings
cargo test --workspace --tests    # CI runs this; locally prefer a focused -p <crate>
cargo check -p koharu -p koharu-app -p koharu-desktop     # focused check for app/IPC changes
cargo test -p koharu-scene <test_name>                    # single test

# Frontend
bun run --filter @koharu/app lint          # oxlint (CI)
bun run --filter @koharu/app test          # vitest run, jsdom (CI)
cd packages/koharu && bun x vitest run tests/lib/<file>.test.ts   # single test file
bun x tsc --noEmit -p packages/koharu/tsconfig.json                 # type-check the app
bun run --filter @koharu/bridge typecheck  # same for bridge/ui packages
bun run format                             # oxfmt (TS/JSON); `bun run check` verifies

# Codegen
cargo run -p koharu-app --bin generate     # regenerate packages/bridge/src/protocol.ts
bun run --filter @koharu/bridge build      # wasm-pack koharu-canvas -> packages/bridge/src/wasm
```

ML models each have a CLI in `crates/koharu-ml/src/bin/<model>.rs`, for example `cargo run -p koharu-ml --bin lama -- -i in.png -m mask.png -o out.png [--cpu]`.

## Architecture

Crate READMEs are authoritative and detailed. Read the one for any crate you touch.

- `koharu` is the binary: startup, diagnostics (tracing, sentry, panic), and Tauri config (`tauri*.conf.json`, `capabilities/`). It composes `koharu-app` and `koharu-desktop`.
- `koharu-app` holds Tauri-managed state and one named command per operation (`src/commands/`). Mutating commands take a project id and the current revision, and the frontend serializes those mutations. Updates go out on independent typed channels, with no shared event envelope. **Rust command signatures are the frontend contract.** tauri-specta generates `packages/bridge/src/protocol.ts` from them.
- `koharu-scene` is the canonical in-memory document. It has three layers: a kernel (identity, hierarchy, revisioned components, patches, undo), a document schema, and an editor facade. `koharu-storage` persists only opaque snapshots plus BLAKE3 content-addressed blobs in a `.khrproj` directory.
- `koharu-pipeline` runs detect → OCR → translate → inpaint, one stage per page. It commits each stage through the caller's `Committer`, so stale or overlapping writes fail conflict validation.
- Rendering: `koharu-renderer` turns a scene page into a Vello frame. `koharu-rasterizer` is the backend-neutral display list and Vello/WGPU compositor, shared by native export and the browser. `koharu-desktop` prepares and publishes frames (latest request wins). `koharu-canvas` is **wasm32-only** and runs in the browser as the viewport. `koharu-psd` handles PSD export.
- `koharu-ml` has one module per model, usually split into `config.rs`, `model.rs` and `processor.rs`. `crate::model_repository!` pins weights to a Hugging Face repo at a specific commit. The public lifecycle is `Model::load(device).await` followed by `inference(...)` wrapped in `koharu_torch::no_grad`.
- The native layers follow the `-sys` + safe-wrapper pattern:
  - `koharu-torch(-sys)`, `koharu-llama(-sys)`, `koharu-diffusion(-sys)` split each library into raw bindings and a safe API.
  - `koharu-bindgen` generates the dynamic-loading bindings from `build.rs`.
  - `koharu-runtime` discovers, downloads and loads the native runtime packages. They are cached under the OS cache dir.
- `koharu-translator` holds local GGUF and hosted provider backends, and each provider owns its own config and defaults. Credentials live in the OS keychain (`koharu-secrets`), never in config (`koharu-config`). `koharu-agent` orchestrates OAuth-backed Codex agents. `koharu-metrics` turns tracing events into anonymous analytics.
- Frontend packages:
  - `packages/koharu` (`@koharu/app`) is the Next.js app. It has `app/`, `components/`, `lib/` (zustand store, react-query queries, backend helpers), and vitest tests under `tests/`.
  - `packages/ui` is the shared shadcn component library (`bun run shadcn` overwrites it).
  - `packages/bridge` contains `protocol.ts` (generated), `canvas.ts` (hand-written adapter) and `src/wasm` (derived, gitignored).
  - `packages/docs` is the Mintlify site (en/ja/zh).
- The responsibility split: React owns tools, hit testing, selection and gestures. Native Rust owns the scene, commits, undo, persistence and frame preparation. The browser never commits project state.

## Conventions

- Rust uses `anyhow` with `.context(...)` in binaries and orchestration. Most library crates define `thiserror` errors. Formatting is default rustfmt with 4-space indents. Clippy must pass with `-D warnings`.
- Rust tests are inline `#[cfg(test)] mod tests` blocks, plus `tests/` in `koharu-renderer` and `koharu-rasterizer`, and criterion `benches/` in scene, storage, renderer and ml. Tests that download checkpoints or need LibTorch use `#[ignore = "reason"]`. Don't run those unless asked.
- TS/TSX style (enforced by oxfmt): no semicolons, single quotes (in JSX too), 2-space indents. Imports are sorted and grouped builtin/external/internal (`@/`, `@koharu/`)/relative, with blank lines between groups. Tailwind classes are sorted inside `cn`/`clsx`/`cva`/`tw`.
- Commits follow Conventional Commits with scopes, for example `fix(renderer): ...`, `feat(ui): ...`, `chore(deps): ...`. git-cliff builds `CHANGELOG.md` from them, and `scripts/release.ts` bumps `[workspace.package].version` (`chore(release): x.y.z`). PRs use the template (Summary + Test plan) and must disclose substantial AI help (see `CONTRIBUTING.md`).
- Put new dependency versions in `[workspace.dependencies]` and reference them with `{ workspace = true }`.

## Gotchas

- **Generated files. Do not edit these by hand; change the source instead:**
  - `packages/bridge/src/protocol.ts`: change the Rust command/type, then run `generate`. oxfmt and oxlint ignore it.
  - `packages/bridge/src/wasm/`: rebuild the bridge.
  - `crates/koharu-torch-sys/libtch/torch_api_generated.h`
  - The `-sys` bindings in `OUT_DIR`.
  - The vendored llama.cpp/stable-diffusion.cpp headers: sync them with `.agents/skills/runtime/scripts/sync.sh` (see `.agents/skills/runtime/SKILL.md`).
- After changing a Tauri command or any type it exposes, regenerate `protocol.ts` and update the TS consumers in the same change.
- Changes to `koharu-canvas` or `koharu-rasterizer` reach the UI only after the bridge WASM is rebuilt. `bun dev` watches these with nodemon. Vitest aliases `koharu_canvas.js` to `tests/mocks/koharu_canvas.ts`, so UI tests never exercise the real WASM.
- Several pins are exact and should move together: `tauri*` and `@tauri-apps/*` (3.0.0-alpha), `specta`/`specta-typescript`, and a forked `tauri-specta` git tag. `hf-hub` tracks git `main`.
- `koharu-canvas` only has real dependencies on `wasm32`. Checking it natively compiles almost nothing, so build it with the bridge build (wasm-pack) instead.
- `next dev` rewrites the Next.js block in `packages/koharu/AGENTS.md`. Commit that diff rather than fighting it. This Next.js version differs from training data, so read `node_modules/next/dist/docs/` before writing Next code.
- Debug builds expose CEF DevTools on `127.0.0.1:4000` (`crates/koharu-app/src/app.rs`).
- Linux builds need GTK 4 and friends (see the `apt install` list in `.github/workflows/lint.yml`). The Rust tests in CI also need `fonts-noto-cjk`, `dbus-x11` and `gnome-keyring`, because renderer tests use CJK fonts and the secrets tests use the keyring.
- `.cargo/config.toml` sets `CARGO_WORKSPACE_DIR` to the repo root. Dev profile builds image/codec/canvas/rasterizer crates at `opt-level = 3`, so the first debug build is slower than expected.

## Open questions

- There is no local wrapper for the CI Rust tests' dependency on the keyring/dbus. It is unclear which tests fail without them on Windows/macOS.
- `bun run format` targets a root `tests/` directory that does not exist. It may be a leftover from removed integration tests; the `.gitignore` still has entries for them.
- The TS type-check isn't part of CI or root scripts. `@koharu/app` has no `typecheck` script, so the `tsc -p` command above comes from `crates/koharu/README.md`.
