# ShopClaw Development Guide

## Project Overview

ShopClaw is a local-first AI shopping assistant. It exposes shopping site operations as MCP tools so AI Agents can search, compare prices, and manage carts in the user's own browser.

**Language**: Rust (core + plugins)

**Browser Extension**: ShopClaw depends on the KleePay Browser Relay extension (repo: [Edmonds-LR/kleepay-browser-relay](https://github.com/Edmonds-LR/kleepay-browser-relay)). ShopClaw does not ship its own extension.

## Repository Structure

```
ShopClaw/
├── docs/               # All design documents live here
│   ├── DESIGN.md       # Full technical design (architecture, types, Rust code)
│   └── REVIEW.md       # Design review findings and open issues
├── registry/           # Selector hot-update source (GitHub Raw)
│   ├── manifest.json   # Global version manifest
│   ├── amazon/         # Amazon selectors + meta
│   ├── jd/             # JD selectors + meta
│   └── taobao/         # Taobao selectors + meta
├── crates/             # Rust workspace (when code exists)
│   ├── shopclaw-core/  # MCP host, plugin manager, browser bridge
│   ├── shopclaw-plugin-sdk/  # Plugin SDK (compiled by plugins, not itself)
│   └── shopclaw-plugin-*/    # Individual site plugins (compile to WASM)
├── README.md           # User-facing overview
├── CLAUDE.md           # This file
└── .gitignore
```

## Key Design Decisions

1. **Plugin sandbox = WASM (Wasmtime)** — not Deno. Plugins are Rust compiled to wasm32-wasi.
2. **Plugin interface = WIT (Component Model)** — not raw `extern "C"` FFI. Type-safe, no unsafe code in plugins.
3. **Selectors are decoupled from plugins** — stored in `registry/`, hot-updated from this repo via GitHub Raw. Plugins request selectors from the host at runtime.
4. **Browser control via KleePay Browser Relay extension** — not `--remote-debugging-port`. The KleePay extension (shared with KleePay Signer) provides CDP relay over localhost WebSocket.
5. **No cloud backend** — everything runs locally. Selector sync uses GitHub Raw (no custom server).

## Development Conventions

- All documentation goes in `docs/`. Do not create .md files in the root except README.md and CLAUDE.md.
- `registry/` contains only JSON files (selectors, manifests, meta). No code.
- Rust code follows standard `cargo fmt` + `cargo clippy`.
- Plugin crates use `crate-type = ["cdylib"]`. The SDK crate uses `crate-type = ["rlib"]`.
- Commit messages: imperative mood, concise first line, details in body if needed.

## Common Tasks

### Adding a new shopping site plugin

1. Create `crates/shopclaw-plugin-{site}/` with `Cargo.toml`, `src/lib.rs`
2. Add `manifest.json` declaring tools and risk levels
3. Add `registry/{site}/selectors.json` with initial selectors
4. Add `registry/{site}/meta.json` with version info
5. Update `registry/manifest.json` to include the new plugin

### Updating selectors for a site

Just edit `registry/{site}/selectors.json` directly. Bump `selector_version` in both `registry/{site}/meta.json` and `registry/manifest.json`. Users will auto-sync on next check.

### Design documents

- `docs/DESIGN.md` — full technical design with architecture diagrams, Rust code, and WIT interfaces
- `docs/REVIEW.md` — design review findings, open security issues, and coverage gaps
