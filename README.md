# ShopClaw

Local AI shopping assistant that works with any MCP-compatible AI Agent.

ShopClaw runs on your machine, operates your own browser, and never touches your passwords or payment details. Think of it as a smarter Honey — instead of just finding coupons, it can search products, compare prices across sites, and manage your cart, all driven by natural language through your AI Agent.

## How It Works

```
You: "Compare AirPods prices on Amazon and JD"

         ┌──────────────┐     MCP      ┌──────────────┐     CDP      ┌──────────────┐
         │   AI Agent   │ ──────────── │   ShopClaw   │ ──────────── │ Your Chrome  │
         │  (OpenClaw,  │   stdio      │  (Rust, local│  KleePay     │  (your login │
         │  Claude Code)│              │   process)   │  Browser     │   sessions)  │
         └──────────────┘              └──────────────┘  Relay ext.  └──────────────┘

Agent: "Here's what I found:
        | Product      | Amazon  | JD     |
        | AirPods Pro  | $249    | ¥1,799 |
        | AirPods 3    | $169    | ¥1,279 |
        Amazon is cheaper. Want me to add to cart?"
```

## Features

- **Cross-site search & compare** — search Amazon, JD, Taobao and more in one command
- **Smart cart management** — add to cart, view cart, track prices
- **Plugin system** — add support for new sites via WASM plugins
- **Selector hot-update** — sites change their UI? Selectors update automatically, no reinstall needed
- **LLM fallback** — if selectors break before an update is available, LLM figures out the new page structure

## Safety by Design

| Principle | Implementation |
|-----------|---------------|
| **No credentials** | Operates your browser directly — never stores your passwords |
| **No auto-payment** | Cart operations only; checkout requires your explicit confirmation |
| **No anti-bot bypass** | Uses your real browser; captchas are handed to you |
| **No cloud** | Runs locally, no backend, no telemetry, no data upload |
| **Sandboxed plugins** | Each plugin runs in a WASM sandbox — no file system, no network |

## Quick Start

```bash
# Prerequisites: install KleePay Browser Relay extension from Chrome Web Store
# https://github.com/Edmonds-LR/kleepay-browser-relay

# Install
cargo install shopclaw

# First-time setup (checks KleePay Browser Relay extension, creates config)
shopclaw setup

# Start as MCP server (AI Agent connects via stdio)
shopclaw serve
```

### Agent Configuration

```json
// ~/.openclaw/mcp.json (or any MCP-compatible agent)
{
  "mcpServers": {
    "shopclaw": {
      "command": "shopclaw",
      "args": ["serve"]
    }
  }
}
```

Once configured, your Agent automatically discovers ShopClaw's tools:

| Tool | What it does |
|------|-------------|
| `amazon_search` | Search products on Amazon |
| `amazon_add_to_cart` | Add a product to your Amazon cart |
| `amazon_view_cart` | View your Amazon cart |
| `jd_search` | Search products on JD.com |
| `jd_add_to_cart` | Add a product to your JD cart |
| `taobao_search` | Search products on Taobao |
| ... | More tools from more plugins |

## Architecture

```
ShopClaw (Rust binary)
├── MCP Server          # stdio JSON-RPC 2.0, talks to AI Agent
├── Plugin Router       # Routes tool calls to the right plugin
│   ├── Confirmation Gate   # Blocks sensitive operations without user OK
│   ├── Rate Limiter        # Prevents triggering site anti-bot
│   └── Audit Logger        # Logs all operations locally
├── Selector Resolver   # 3-layer: local cache → GitHub sync → LLM fallback
├── Browser Bridge      # WebSocket → KleePay Browser Relay extension → CDP
└── Plugin Sandbox      # Wasmtime WASM runtime
    ├── Amazon Plugin (.wasm)
    ├── JD Plugin (.wasm)
    └── Taobao Plugin (.wasm)
```

See [docs/DESIGN.md](docs/DESIGN.md) for the full technical design with architecture diagrams, Rust types, and WIT interfaces.

## Plugin System

Plugins are written in Rust and compiled to WebAssembly. Each plugin runs in an isolated WASM sandbox with no access to files, network, or other plugins.

Plugins interact with the browser through a host-provided API (WIT interface):

```rust
// Plugin code — type-safe, no unsafe needed
let tab = browser::open_tab("https://amazon.com/s?k=airpods")?;
let sel = selectors::get_or_llm_fallback("result_title", "Product title")?;
let title = browser::query_text(tab.id, &sel)?;
browser::close_tab(tab.id);
```

### Writing a Plugin

See [docs/DESIGN.md#4-plugin-system-wasm-sandbox](docs/DESIGN.md#4-plugin-system-wasm-sandbox) for the full plugin development guide.

## Selector Hot-Update

Shopping sites change their HTML frequently. ShopClaw handles this automatically:

```
Selector fails? (site changed its UI)
  │
  ├─ Layer 1: Check local cache → maybe already updated
  ├─ Layer 2: Pull latest from GitHub → community keeps selectors current
  └─ Layer 3: LLM analyzes the page → discovers new selector automatically
```

Selectors live in [`registry/`](registry/) and are synced every 6 hours via GitHub Raw. No reinstall needed.

## CLI Commands

```bash
shopclaw serve                    # Start MCP server
shopclaw setup                    # First-time setup wizard
shopclaw plugins list             # List installed plugins
shopclaw plugins install amazon   # Install a plugin
shopclaw plugins remove jd        # Remove a plugin
shopclaw audit                    # View recent operations
shopclaw audit -n 50 --plugin amazon  # Filter audit log
```

## Configuration

```toml
# ~/.shopclaw/config.toml

[browser]
max_concurrent_tabs = 3
navigation_timeout_ms = 15000

[rate_limit]
search_per_min = 10
cart_per_min = 5

[confirmation]
mode = "native_dialog"    # or "terminal"

[selector_sync]
sync_interval = "6h"
registry_url = "https://raw.githubusercontent.com/tokligence/ShopClaw/main/registry"

[llm]
provider = "anthropic"    # or "local" (Ollama), "openai"
model = "claude-sonnet-4-20250514"
```

## Legal

ShopClaw is a local browser automation tool that operates within the user's own browser environment. It does not access websites on behalf of users, store credentials, or bypass security measures.

Users are responsible for ensuring their use of ShopClaw complies with the terms of service of any websites they interact with. ShopClaw is provided "as-is" without warranty.

## License

Apache-2.0
