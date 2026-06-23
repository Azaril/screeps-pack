# screeps-pack

> npm-free build + deploy for Rust [Screeps](https://screeps.com) bots — `cargo build` to running code on any server, no Node toolchain.

`screeps-pack` takes a Rust Screeps bot crate from `cargo build` all the way to code running on a server, without npm, Node, webpack, rollup, or a starter clone. It runs `cargo build --target wasm32-unknown-unknown`, fetches the exact `wasm-bindgen` your `Cargo.lock` pins, patches the generated glue for the Screeps isolate, optionally runs `wasm-opt`, and uploads the resulting module map to any server entry in a `.screeps.yaml` credentials file. It ships as both a library and a thin CLI: the same pipeline functions other harnesses call programmatically are what the `screeps-pack` binary drives. It was extracted from the [screeps-ibex](https://github.com/Azaril/screeps-ibex) workspace, where it replaced an npm-based `deploy.js` pipeline.

## Installation

`screeps-pack` is a CLI and a library.

Install the CLI from git:

```bash
cargo install --git https://github.com/Azaril/screeps-pack
```

Or run it from a source checkout (handy for the in-workspace layout, where `screeps-pack` sits next to your bot crate):

```bash
cargo run --manifest-path screeps-pack/Cargo.toml -- deploy --server mmo
```

Use it as a library by adding the git dependency:

```toml
[dependencies]
screeps-pack = { git = "https://github.com/Azaril/screeps-pack" }
```

## Getting Started

### Prerequisites

- **Rust with the `wasm32-unknown-unknown` target:**

  ```bash
  rustup target add wasm32-unknown-unknown
  ```

  If your build flags use `-Z build-std` (common for Screeps — see the `.screeps.yaml` example below), you also need a **nightly** toolchain plus the `rust-src` component. Pin both in a `rust-toolchain.toml` in your bot crate. `screeps-pack` sanity-checks `cargo --version` up front when `-Z` flags are configured and gives a clear error if nightly is missing.

- **A Screeps bot crate** with `crate-type = ["cdylib", ...]` that links `wasm-bindgen` and exports `setup()` (called once after load) and `game_loop()` (called per tick) — the [screeps-starter-rust](https://github.com/rustyscreeps/screeps-starter-rust) convention. No `js_src/`, `package.json`, or bundler config is needed.

- **A `.screeps.yaml` credentials file** ([SS3 unified credentials format](https://github.com/screepers/screepers-standards/blob/master/SS3-Unified_Credentials_File.md), shared by screepers tooling). A minimal one:

  ```yaml
  servers:
    mmo:
      host: screeps.com
      secure: true
      token: your-auth-token        # official servers: token auth
    private-server:
      host: 127.0.0.1
      port: 21025
      username: you
      password: your-password       # private servers: signin auth

  configs:
    # Extra args appended to `cargo build`, resolved per server.
    # '*' applies everywhere; per-server entries are concatenated AFTER it.
    wasm-pack-options:
      '*': ["--config", "build.rustflags=['-Ctarget-cpu=mvp']", "-Z", "build-std=std,panic_abort"]
      mmo: ["--features", "mmo"]
  ```

  The `-Ctarget-cpu=mvp` + `build-std` combination keeps the emitted wasm inside the MVP feature set older Screeps server Node versions require. The `configs.wasm-pack-options` section is honored exactly as the classic `deploy.js`/wasm-pack pipeline resolved it, so existing configs are drop-in (the historical name is kept for compatibility; the args go straight to `cargo build`).

### Your first build and deploy

From your bot crate's directory (or its workspace root — a virtual workspace resolves to its single cdylib member automatically):

1. **Preview the plan — builds nothing, uploads nothing:**

   ```bash
   screeps-pack check --server private-server
   ```

   This prints the resolved server entry, the bot crate, the exact `cargo` argv for both release and debug, and the `wasm-bindgen` version resolved from your lockfile (plus the archive URL it would fetch). Secrets never print. Use it to confirm everything resolves before a slow build.

2. **Build the full module map locally — no upload:**

   ```bash
   screeps-pack build --server private-server
   ```

   On the first run this downloads and caches the lockfile-matched `wasm-bindgen` CLI and the pinned binaryen `wasm-opt` under `<target>/screeps-pack/tools/`. The artifacts (three modules plus a `manifest.json` of sizes and SHA-256 hashes) land under `<target>/screeps-pack/dist/<server>/<mode>/`.

3. **Build and upload:**

   ```bash
   screeps-pack deploy --server private-server
   ```

   This runs the full pipeline and uploads the module map to the entry's branch via `POST /api/user/code`. On success it prints the uploaded module names, sizes, hashes, the build duration, and the fraction of the 5 MiB code-size limit used.

## Common Use Cases

**Deploy to a named server (the canonical invocation).** Pick any `servers:` entry by name:

```bash
screeps-pack deploy --server mmo
```

In the in-workspace layout, where `screeps-pack` is a sibling crate of the bot, the equivalent is:

```bash
cargo run --manifest-path screeps-pack/Cargo.toml -- deploy --server mmo
```

**Build the bundle without uploading.** Produce the same artifacts `deploy` would push, written to `<target>/screeps-pack/dist/<server>/<mode>/`, but upload nothing — useful for inspecting the module map or diffing against another pipeline:

```bash
screeps-pack build --server mmo
```

`deploy --dryrun` does the same as a one-off (build everything, print the module map, skip the upload):

```bash
screeps-pack deploy --server mmo --dryrun
```

**Deploy to a private server vs. MMO.** The auth method follows the entry: official servers (`mmo`, `ptr`, season) use `token:`; private servers use `username:` + `password:` (token wins if both are present). Nothing about the command changes — only the entry name:

```bash
screeps-pack deploy --server private-server   # username/password signin
screeps-pack deploy --server mmo              # token auth
```

Private-server entries default to port `21025` and plain HTTP; `secure: true` entries default to port `443` over HTTPS.

**Make a fast debug build.** Drop `--release` (the wasm-pack `--dev` equivalent), which also selects the dev `wasm-opt` profile. Debug builds are much larger and commonly exceed the 5 MiB code limit on big bots — that is the build mode, not the tool:

```bash
screeps-pack deploy --server private-server --debug
```

**Point at a crate or credentials file elsewhere.** Both default to the invocation directory but can be overridden (these flags are global and work with any subcommand):

```bash
screeps-pack deploy --server mmo \
  --manifest-path ../my-bot/Cargo.toml \
  --creds ~/.config/screeps/.screeps.yaml
```

**See where tools are cached.** The lockfile-matched `wasm-bindgen` CLI and the pinned binaryen `wasm-opt` are downloaded from GitHub releases on first use and cached under your build's cargo target directory at `<target>/screeps-pack/tools/`. The cache is per-project, content-named by version, and survives `cargo clean -p`; subsequent builds reuse it with no network access. If `wasm-opt` can't be downloaded (blocked network, unsupported platform), `screeps-pack` falls back to a `wasm-opt` on `PATH`, and if neither exists it skips optimization with a warning (the un-optimized wasm is valid, just larger).

## Usage

As a library, the entry points are `build` (assemble the module map, no upload) and `deploy` (build and upload), both driven by a `PackOptions`:

```rust
use screeps_pack::{deploy, PackOptions};
use std::path::PathBuf;

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let options = PackOptions {
        // The bot crate's Cargo.toml, or a workspace root with one cdylib member.
        manifest_path: PathBuf::from("Cargo.toml"),
        // The SS3 .screeps.yaml credentials file.
        creds_path: PathBuf::from(".screeps.yaml"),
        // The `servers:` entry to build flags from and upload to.
        server: "private-server".to_string(),
        // Debug build (no --release; the dev wasm-opt profile).
        debug: false,
    };

    // `deploy` builds + uploads; `screeps_pack::build` stops before the upload.
    let outcome = deploy(&options).await?;
    println!("{outcome}"); // Display: server, mode, modules, sizes, hashes, timing
    Ok(())
}
```

`deploy` (and `build`) return a `PackOutcome` with the resolved server/branch, build mode, `wasm-bindgen` version, per-module info (`Vec<ModuleInfo>`: name, kind, byte size, SHA-256), the size used against the 5 MiB limit, the dist directory, and the run duration. It also implements `Display` for a ready-made summary.

Lower-level steps are available as public modules if you need to drive part of the pipeline yourself: `config` (`.screeps.yaml` parsing — `load_server_config`, `ServerConfig`, `ServerAuth`), `project` (bot-crate resolution — `resolve_project`, `Project`), `cargo_build`, `bindgen`, `glue`, `opt`, `tools` (version-locked tool download/cache), and `upload` (module-map assembly + `POST /api/user/code`).

## CLI

All subcommands take a required `--server <entry>` naming a `servers:` entry in the credentials file. Two global flags work with any subcommand:

| Flag | Default | Meaning |
|---|---|---|
| `--manifest-path <path>` | `./Cargo.toml` (a workspace root resolves to its single cdylib member) | the bot crate to build |
| `--creds <path>` | `./.screeps.yaml` | the SS3 credentials file |

### `deploy --server <entry>`

Build and upload to the entry's branch.

- `--debug` — debug build (no `--release`; the dev `wasm-opt` profile).
- `--dryrun` — build everything and print the module map, but upload nothing.

### `build --server <entry>`

Build the full module map without uploading — the same artifacts `deploy` would push, written under `<target>/screeps-pack/dist/<server>/<mode>/`.

- `--debug` — as above.

### `check --server <entry>`

Resolve and print the plan — server entry, bot crate, `cargo` argv (release and debug), and lockfile-resolved `wasm-bindgen` version — with no build and no upload. Secrets are never printed.

Logging verbosity is controlled by the `RUST_LOG` environment variable (e.g. `RUST_LOG=screeps_pack=debug`); it defaults to `screeps_pack=info`.

## How it works

The pipeline is seven steps, each a library module; the CLI in `main.rs` is a thin [clap](https://github.com/clap-rs/clap) wrapper over them:

1. **config** — parse `.screeps.yaml` and resolve the chosen server entry, including `configs.wasm-pack-options` (the `'*'`-then-per-server arg concatenation, passed through to `cargo build` verbatim). Tokens and passwords live in `secrecy::SecretString` from the parse boundary; `Debug` redaction is test-pinned, so nothing secret reaches logs, dist files, or `manifest.json`.
2. **project** — resolve the bot crate (a direct `[package]` manifest, or the single cdylib member of a virtual workspace) and read its metadata knobs: the existing `[package.metadata.wasm-pack.profile.*] wasm-opt` tables (honored as-is for drop-in parity) and `[package.metadata.screeps-pack] bucket-boot-threshold` (the loader's boot gate, default 1500).
3. **cargo_build** — `cargo build --lib --target wasm32-unknown-unknown` with the per-server args; the cdylib `.wasm` is located from cargo's `--message-format=json` stream rather than by guessing the target-dir layout.
4. **bindgen** — resolve the `wasm-bindgen` version from the bot's `Cargo.lock`, fetch the *matching* prebuilt CLI, and run it with `--target nodejs`. The version is never a static pin: the bindgen schema embedded in the wasm must match the CLI exactly (wasm-bindgen#1587), so the lockfile decides and the tool follows.
5. **glue** — patch the nodejs-target bindgen output for the Screeps isolate. The patch is *anchored* on the exact emitted shape and hard-errors (naming the version) on anything it hasn't been verified against, rather than risk silent corruption. A vendored CC0 TextEncoder/TextDecoder polyfill is prepended (the isolate has neither), and the synchronous `fs`/`__dirname` wasm load is replaced by a deferred `__instantiate` under loader control. The loader (`main`) module — bucket-gated boot, staged multi-tick wasm init, a `console.error` shim, and the [wasm-bindgen#3130](https://github.com/rustwasm/wasm-bindgen/issues/3130) `Game.cpu.halt()` trap — is rendered from an embedded template.
6. **opt** — optionally run `wasm-opt` (binaryen) with the args from the crate's wasm-pack profile metadata. The pinned binaryen release is downloaded for byte-parity with wasm-pack; a `PATH` `wasm-opt` is the fallback; if neither is available the step is skipped gracefully.
7. **upload** — assemble the module map and `POST /api/user/code` via the shared `screeps-rest-api` client (token or username/password). Unlike a bundler pipeline's two modules, this uploads three (`main`, `<module_name>`, `<module_name>_bg`, where `<module_name>` is the crate name with hyphens replaced by underscores — e.g. `screeps-ibex` ⇒ `screeps_ibex`): the bindgen glue is its own CommonJS module rather than being inlined, which the engine's native `require()` of `{binary}` modules handles directly.

The design rationale — why `--target nodejs` over `--target web` plus a bundler, and the byte-for-byte parity evidence against the old `deploy.js` pipeline — is recorded in [PARITY.md](PARITY.md).

## Related crates

- [screeps-rest-api](https://github.com/Azaril/screeps-rest-api) — the shared Screeps REST client (token and username/password auth, the `POST /api/user/code` upload shape) this crate depends on for upload.
- [screeps-server-kit](https://github.com/Azaril/screeps-server-kit) — a private-server / harness toolkit that drives the `screeps-pack` library functions programmatically.
- [screeps-prospector](https://github.com/Azaril/screeps-prospector) — a sibling tool from the same workspace for Screeps room/world analysis.
