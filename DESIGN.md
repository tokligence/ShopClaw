# ShopClaw — Technical Design Document

## 1. Project Overview

ShopClaw is a **local-first AI shopping assistant** that exposes shopping site operations as MCP (Model Context Protocol) tools. AI Agents discover and invoke these tools to search products, compare prices, and manage carts — all within the user's own browser.

**Language**: Rust (core + plugins), TypeScript (Chrome extension only)

**Key constraints**:
- No cloud backend, no telemetry, no credential storage
- No anti-bot circumvention; graceful degradation on captcha/risk-control
- No automatic payment; sensitive operations require user confirmation
- Plugin sandboxing via WebAssembly (Wasmtime)

## 2. Workspace Layout

```
ShopClaw/
├── Cargo.toml                    # Workspace root
├── DESIGN.md                     # This file
├── LICENSE-MIT
├── LICENSE-APACHE
│
├── crates/
│   ├── shopclaw-core/            # MCP host, plugin manager, config
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── main.rs           # CLI entrypoint
│   │       ├── mcp/              # MCP protocol (stdio JSON-RPC 2.0)
│   │       │   ├── mod.rs
│   │       │   ├── server.rs     # MCP server loop
│   │       │   ├── types.rs      # Tool, ToolResult, etc.
│   │       │   └── router.rs     # Route tool calls to plugins
│   │       ├── plugin/           # Plugin lifecycle
│   │       │   ├── mod.rs
│   │       │   ├── manifest.rs   # Parse manifest.json
│   │       │   ├── registry.rs   # Hot-load/unload plugins
│   │       │   └── sandbox.rs    # Wasmtime runtime
│   │       ├── browser/          # Browser Bridge
│   │       │   ├── mod.rs
│   │       │   ├── relay.rs      # WebSocket client → Chrome extension
│   │       │   ├── cdp.rs        # CDP command builder
│   │       │   └── tab.rs        # Tab abstraction
│   │       ├── gate.rs           # Confirmation Gate
│   │       ├── limiter.rs        # Rate Limiter (token bucket)
│   │       ├── audit.rs          # Operation Logger
│   │       └── config.rs         # Config file (~/.shopclaw/config.toml)
│   │
│   ├── shopclaw-plugin-sdk/      # Plugin SDK (compiled to WASM)
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs            # #[shopclaw_tool] macro, host imports
│   │       ├── types.rs          # Product, CartItem, SearchResult...
│   │       ├── bridge.rs         # Host-provided browser API (FFI)
│   │       └── selector.rs       # CSS/XPath selector helpers
│   │
│   ├── shopclaw-plugin-amazon/   # Amazon plugin (WASM)
│   │   ├── Cargo.toml
│   │   ├── manifest.json
│   │   ├── selectors.json
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── search.rs
│   │       ├── product.rs
│   │       ├── cart.rs
│   │       └── price_track.rs
│   │
│   ├── shopclaw-plugin-jd/       # JD.com plugin (WASM)
│   │   ├── Cargo.toml
│   │   ├── manifest.json
│   │   ├── selectors.json
│   │   └── src/
│   │       └── lib.rs
│   │
│   └── shopclaw-plugin-taobao/   # Taobao plugin (WASM)
│       ├── Cargo.toml
│       ├── manifest.json
│       ├── selectors.json
│       └── src/
│           └── lib.rs
│
├── extension/                    # Chrome extension (TypeScript)
│   ├── manifest.json             # Chrome MV3 manifest
│   ├── src/
│   │   ├── background.ts        # Service worker: WebSocket server + CDP relay
│   │   ├── popup.ts             # Connection status + disconnect button
│   │   └── content.ts           # (minimal) page-level hooks if needed
│   ├── package.json
│   └── tsconfig.json
│
├── tests/
│   ├── integration/              # End-to-end tests with headless Chrome
│   └── fixtures/                 # Mock HTML pages for plugin tests
│
└── docs/
    ├── plugin-dev-guide.md       # How to write a ShopClaw plugin
    └── architecture.md           # High-level architecture diagrams
```

## 3. Core Architecture

### 3.1 Component Diagram

```
                          stdio (JSON-RPC 2.0)
AI Agent ◄──────────────────────────────────────────► MCP Server
                                                         │
                                                         │ route tool call
                                                         ▼
                                                    ┌─────────┐
                                                    │  Router  │
                                                    └────┬────┘
                                    ┌────────────────────┼────────────────────┐
                                    ▼                    ▼                    ▼
                              ┌───────────┐       ┌───────────┐       ┌───────────┐
                              │  Amazon   │       │    JD     │       │  Taobao   │
                              │  Plugin   │       │  Plugin   │       │  Plugin   │
                              │  (WASM)   │       │  (WASM)   │       │  (WASM)   │
                              └─────┬─────┘       └─────┬─────┘       └─────┬─────┘
                                    │ host calls        │                    │
                                    ▼                    ▼                    ▼
          ┌──────────────────────────────────────────────────────────────────────┐
          │                         Host API Layer                              │
          │  ┌──────────┐  ┌──────────────┐  ┌───────────┐  ┌──────────────┐  │
          │  │ Browser  │  │ Confirmation │  │   Rate    │  │   Audit     │  │
          │  │ Bridge   │  │    Gate      │  │  Limiter  │  │   Logger    │  │
          │  └────┬─────┘  └──────────────┘  └───────────┘  └──────────────┘  │
          └───────┼──────────────────────────────────────────────────────────────┘
                  │ WebSocket (ws://127.0.0.1:PORT)
                  ▼
          ┌──────────────┐
          │ Chrome Ext.  │ ── chrome.debugger API ──► User's Chrome Tabs
          │ (Browser     │
          │  Relay)      │
          └──────────────┘
```

### 3.2 Core Traits

```rust
// crates/shopclaw-core/src/mcp/types.rs

use serde::{Deserialize, Serialize};

/// Risk level determines whether user confirmation is required.
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
#[serde(rename_all = "snake_case")]
pub enum RiskLevel {
    /// Read-only, no side effects (search, view details)
    Read,
    /// Modifies state but no financial impact (add to cart)
    Write,
    /// Financial or irreversible action (checkout) — requires confirmation
    Sensitive,
    /// Absolutely forbidden (complete payment, change password)
    Blocked,
}

/// A tool exposed via MCP, discovered by the AI Agent.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ToolDefinition {
    pub name: String,
    pub description: String,
    pub risk_level: RiskLevel,
    pub parameters: serde_json::Value, // JSON Schema
    pub requires_confirmation: bool,
}

/// Result returned to the Agent after a tool invocation.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ToolResult {
    pub success: bool,
    pub data: serde_json::Value,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub error: Option<String>,
    /// If true, the operation was blocked and needs user action.
    #[serde(default)]
    pub needs_user_action: bool,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub user_action_prompt: Option<String>,
}

/// MCP JSON-RPC request/response types.
#[derive(Debug, Serialize, Deserialize)]
pub struct JsonRpcRequest {
    pub jsonrpc: String, // "2.0"
    pub id: Option<serde_json::Value>,
    pub method: String,
    pub params: Option<serde_json::Value>,
}

#[derive(Debug, Serialize, Deserialize)]
pub struct JsonRpcResponse {
    pub jsonrpc: String,
    pub id: Option<serde_json::Value>,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub result: Option<serde_json::Value>,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub error: Option<JsonRpcError>,
}

#[derive(Debug, Serialize, Deserialize)]
pub struct JsonRpcError {
    pub code: i32,
    pub message: String,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub data: Option<serde_json::Value>,
}
```

```rust
// crates/shopclaw-core/src/plugin/mod.rs

use crate::mcp::types::{ToolDefinition, ToolResult};
use async_trait::async_trait;

/// Plugin metadata parsed from manifest.json.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct PluginManifest {
    pub name: String,
    pub version: String,
    pub site: String,
    pub supported_regions: Vec<String>,
    pub tools: Vec<ToolDefinition>,
    pub permissions: Vec<String>,
    pub blocked_actions: Vec<String>,
}

/// Trait that every plugin must implement.
/// Plugins are compiled to WASM and loaded by the host.
#[async_trait]
pub trait Plugin: Send + Sync {
    /// Return the manifest describing this plugin's tools.
    fn manifest(&self) -> &PluginManifest;

    /// Invoke a tool by name with the given parameters.
    /// The plugin calls host-provided Browser Bridge APIs internally.
    async fn invoke(
        &self,
        tool_name: &str,
        params: serde_json::Value,
    ) -> Result<ToolResult, PluginError>;
}

#[derive(Debug, thiserror::Error)]
pub enum PluginError {
    #[error("tool not found: {0}")]
    ToolNotFound(String),
    #[error("browser operation failed: {0}")]
    BrowserError(String),
    #[error("selector not found for element: {0}")]
    SelectorNotFound(String),
    #[error("operation blocked by risk policy: {0}")]
    Blocked(String),
    #[error("rate limited: retry after {retry_after_secs}s")]
    RateLimited { retry_after_secs: u64 },
    #[error("user action required: {0}")]
    UserActionRequired(String),
    #[error(transparent)]
    Other(#[from] anyhow::Error),
}
```

### 3.3 Browser Bridge

```rust
// crates/shopclaw-core/src/browser/mod.rs

use serde::{Deserialize, Serialize};

/// Represents a browser tab that can be operated on.
#[derive(Debug, Clone)]
pub struct Tab {
    pub id: String,
    pub url: String,
    pub title: String,
}

/// High-level browser operations provided to plugins.
/// This is the host-side API that WASM plugins call via imported functions.
#[async_trait]
pub trait BrowserBridge: Send + Sync {
    /// Open a new tab and navigate to the given URL.
    async fn open_tab(&self, url: &str) -> Result<Tab, BrowserError>;

    /// Close a tab by ID.
    async fn close_tab(&self, tab_id: &str) -> Result<(), BrowserError>;

    /// Navigate an existing tab to a new URL.
    async fn navigate(&self, tab_id: &str, url: &str) -> Result<(), BrowserError>;

    /// Wait for a CSS selector to appear in the DOM (with timeout).
    async fn wait_for_selector(
        &self,
        tab_id: &str,
        selector: &str,
        timeout_ms: u64,
    ) -> Result<bool, BrowserError>;

    /// Query the DOM for elements matching a CSS selector.
    /// Returns a list of element handles (opaque IDs).
    async fn query_selector_all(
        &self,
        tab_id: &str,
        selector: &str,
    ) -> Result<Vec<ElementHandle>, BrowserError>;

    /// Get text content of an element.
    async fn get_text(&self, tab_id: &str, element: &ElementHandle) -> Result<String, BrowserError>;

    /// Get an attribute value of an element.
    async fn get_attribute(
        &self,
        tab_id: &str,
        element: &ElementHandle,
        attr: &str,
    ) -> Result<Option<String>, BrowserError>;

    /// Click an element.
    async fn click(&self, tab_id: &str, element: &ElementHandle) -> Result<(), BrowserError>;

    /// Type text into an input element.
    async fn type_text(
        &self,
        tab_id: &str,
        element: &ElementHandle,
        text: &str,
    ) -> Result<(), BrowserError>;

    /// Take a screenshot of the tab (for LLM fallback).
    /// Returns PNG bytes.
    async fn screenshot(&self, tab_id: &str) -> Result<Vec<u8>, BrowserError>;

    /// Get the page's full HTML (for LLM fallback).
    async fn get_html(&self, tab_id: &str) -> Result<String, BrowserError>;

    /// Execute JavaScript in the tab and return the result as JSON.
    async fn evaluate(
        &self,
        tab_id: &str,
        expression: &str,
    ) -> Result<serde_json::Value, BrowserError>;

    /// Get the current URL of a tab.
    async fn get_url(&self, tab_id: &str) -> Result<String, BrowserError>;
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ElementHandle {
    pub id: String,
    pub tag: String,
    pub class: Option<String>,
}

#[derive(Debug, thiserror::Error)]
pub enum BrowserError {
    #[error("extension not connected")]
    NotConnected,
    #[error("tab not found: {0}")]
    TabNotFound(String),
    #[error("selector timeout: {0}")]
    SelectorTimeout(String),
    #[error("CDP error: {code} {message}")]
    CdpError { code: i32, message: String },
    #[error("user denied tab operation")]
    UserDenied,
    #[error("websocket error: {0}")]
    WebSocket(String),
}
```

### 3.4 Browser Relay (WebSocket Client)

```rust
// crates/shopclaw-core/src/browser/relay.rs

use tokio::net::TcpStream;
use tokio_tungstenite::{connect_async, MaybeTlsStream, WebSocketStream};
use std::path::PathBuf;

/// Manages the WebSocket connection to the Chrome extension.
pub struct RelayClient {
    ws: Option<WebSocketStream<MaybeTlsStream<TcpStream>>>,
    token: String,
    port: u16,
}

impl RelayClient {
    /// Read the relay token from ~/.shopclaw/relay-token
    pub fn load_token() -> Result<String, BrowserError> {
        let path = dirs::home_dir()
            .unwrap_or_default()
            .join(".shopclaw")
            .join("relay-token");
        std::fs::read_to_string(&path)
            .map(|s| s.trim().to_string())
            .map_err(|_| BrowserError::NotConnected)
    }

    /// Discover the extension's WebSocket port.
    /// The extension writes its port to ~/.shopclaw/relay-port.
    pub fn discover_port() -> Result<u16, BrowserError> {
        let path = dirs::home_dir()
            .unwrap_or_default()
            .join(".shopclaw")
            .join("relay-port");
        let port_str = std::fs::read_to_string(&path)
            .map_err(|_| BrowserError::NotConnected)?;
        port_str.trim().parse::<u16>()
            .map_err(|_| BrowserError::NotConnected)
    }

    /// Connect to the Chrome extension's WebSocket relay.
    pub async fn connect() -> Result<Self, BrowserError> {
        let token = Self::load_token()?;
        let port = Self::discover_port()?;
        let url = format!("ws://127.0.0.1:{}/cdp?token={}", port, token);

        let (ws, _) = connect_async(&url).await
            .map_err(|e| BrowserError::WebSocket(e.to_string()))?;

        Ok(Self { ws: Some(ws), token, port })
    }

    /// Send a CDP command and wait for the response.
    pub async fn send_cdp(
        &mut self,
        method: &str,
        params: serde_json::Value,
    ) -> Result<serde_json::Value, BrowserError> {
        // ... serialize → send → recv → deserialize
        todo!()
    }

    /// Check if the WebSocket connection is alive.
    pub fn is_connected(&self) -> bool {
        self.ws.is_some()
    }
}
```

### 3.5 Confirmation Gate

```rust
// crates/shopclaw-core/src/gate.rs

use crate::mcp::types::RiskLevel;

/// Intercepts tool invocations based on risk level.
/// For `Sensitive` operations, prompts the user for confirmation.
/// For `Blocked` operations, always rejects.
pub struct ConfirmationGate {
    /// How to prompt the user: native OS dialog or terminal prompt.
    mode: ConfirmationMode,
}

pub enum ConfirmationMode {
    /// Use native OS notification + dialog (via `notify-rust` + `native-dialog`).
    NativeDialog,
    /// Print to stderr and read y/n from terminal (for headless/dev mode).
    Terminal,
}

impl ConfirmationGate {
    pub fn new(mode: ConfirmationMode) -> Self {
        Self { mode }
    }

    /// Check if a tool invocation should proceed.
    /// Returns Ok(()) if allowed, Err with reason if blocked/denied.
    pub async fn check(
        &self,
        tool_name: &str,
        risk_level: RiskLevel,
        description: &str,
    ) -> Result<(), GateError> {
        match risk_level {
            RiskLevel::Read | RiskLevel::Write => Ok(()),
            RiskLevel::Sensitive => {
                let approved = self.prompt_user(tool_name, description).await?;
                if approved {
                    Ok(())
                } else {
                    Err(GateError::UserDenied(tool_name.to_string()))
                }
            }
            RiskLevel::Blocked => {
                Err(GateError::Blocked(tool_name.to_string()))
            }
        }
    }

    async fn prompt_user(&self, tool_name: &str, description: &str) -> Result<bool, GateError> {
        match &self.mode {
            ConfirmationMode::NativeDialog => {
                // Use native-dialog crate for OS-level confirmation
                let result = native_dialog::MessageDialog::new()
                    .set_title("ShopClaw Confirmation")
                    .set_text(&format!(
                        "ShopClaw wants to perform a sensitive action:\n\n\
                         Tool: {}\n\
                         Action: {}\n\n\
                         Allow?",
                        tool_name, description
                    ))
                    .set_type(native_dialog::MessageType::Warning)
                    .show_confirm()
                    .unwrap_or(false);
                Ok(result)
            }
            ConfirmationMode::Terminal => {
                eprintln!(
                    "\n[ShopClaw] Sensitive operation: {} — {}\nAllow? (y/N): ",
                    tool_name, description
                );
                let mut input = String::new();
                std::io::stdin().read_line(&mut input).ok();
                Ok(input.trim().eq_ignore_ascii_case("y"))
            }
        }
    }
}

#[derive(Debug, thiserror::Error)]
pub enum GateError {
    #[error("operation blocked by policy: {0}")]
    Blocked(String),
    #[error("user denied operation: {0}")]
    UserDenied(String),
}
```

### 3.6 Rate Limiter

```rust
// crates/shopclaw-core/src/limiter.rs

use std::collections::HashMap;
use std::time::{Duration, Instant};
use tokio::sync::Mutex;

/// Token bucket rate limiter, keyed by (plugin_name, operation_type).
pub struct RateLimiter {
    buckets: Mutex<HashMap<String, TokenBucket>>,
    config: RateLimitConfig,
}

#[derive(Debug, Clone)]
pub struct RateLimitConfig {
    /// search operations: max per minute
    pub search_per_min: u32,
    /// page navigations: max per minute
    pub navigate_per_min: u32,
    /// cart modifications: max per minute
    pub cart_per_min: u32,
    /// price tracking: max per hour per product
    pub price_track_per_hour: u32,
}

impl Default for RateLimitConfig {
    fn default() -> Self {
        Self {
            search_per_min: 10,
            navigate_per_min: 20,
            cart_per_min: 5,
            price_track_per_hour: 1,
        }
    }
}

struct TokenBucket {
    tokens: f64,
    max_tokens: f64,
    refill_rate: f64, // tokens per second
    last_refill: Instant,
}

impl RateLimiter {
    pub fn new(config: RateLimitConfig) -> Self {
        Self {
            buckets: Mutex::new(HashMap::new()),
            config,
        }
    }

    /// Try to consume one token for the given operation.
    /// Returns Ok(()) if allowed, Err with retry-after duration if limited.
    pub async fn check(
        &self,
        plugin: &str,
        operation: &str,
    ) -> Result<(), Duration> {
        let key = format!("{}:{}", plugin, operation);
        let mut buckets = self.buckets.lock().await;

        let bucket = buckets
            .entry(key)
            .or_insert_with(|| self.create_bucket(operation));

        bucket.try_consume()
    }

    fn create_bucket(&self, operation: &str) -> TokenBucket {
        let (max, per_secs) = match operation {
            "search" => (self.config.search_per_min, 60.0),
            "navigate" => (self.config.navigate_per_min, 60.0),
            "cart" => (self.config.cart_per_min, 60.0),
            "price_track" => (self.config.price_track_per_hour, 3600.0),
            _ => (20, 60.0), // default
        };
        TokenBucket::new(max as f64, max as f64 / per_secs)
    }
}

impl TokenBucket {
    fn new(max_tokens: f64, refill_rate: f64) -> Self {
        Self {
            tokens: max_tokens,
            max_tokens,
            refill_rate,
            last_refill: Instant::now(),
        }
    }

    fn try_consume(&mut self) -> Result<(), Duration> {
        let now = Instant::now();
        let elapsed = now.duration_since(self.last_refill).as_secs_f64();
        self.tokens = (self.tokens + elapsed * self.refill_rate).min(self.max_tokens);
        self.last_refill = now;

        if self.tokens >= 1.0 {
            self.tokens -= 1.0;
            Ok(())
        } else {
            let wait = (1.0 - self.tokens) / self.refill_rate;
            Err(Duration::from_secs_f64(wait))
        }
    }
}
```

### 3.7 Audit Logger

```rust
// crates/shopclaw-core/src/audit.rs

use chrono::{DateTime, Utc};
use serde::Serialize;
use std::path::PathBuf;
use tokio::io::AsyncWriteExt;

#[derive(Debug, Serialize)]
pub struct AuditEntry {
    pub timestamp: DateTime<Utc>,
    pub plugin: String,
    pub tool: String,
    pub risk_level: String,
    pub target_url: Option<String>,
    pub action: String,
    pub result: AuditResult,
}

#[derive(Debug, Serialize)]
#[serde(rename_all = "snake_case")]
pub enum AuditResult {
    Success,
    Failed(String),
    Blocked,
    UserDenied,
    RateLimited,
}

pub struct AuditLogger {
    path: PathBuf, // ~/.shopclaw/audit.log
}

impl AuditLogger {
    pub fn new() -> Self {
        let path = dirs::home_dir()
            .unwrap_or_default()
            .join(".shopclaw")
            .join("audit.log");
        Self { path }
    }

    pub async fn log(&self, entry: &AuditEntry) {
        let line = serde_json::to_string(entry).unwrap_or_default();
        if let Ok(mut file) = tokio::fs::OpenOptions::new()
            .create(true)
            .append(true)
            .open(&self.path)
            .await
        {
            let _ = file.write_all(format!("{}\n", line).as_bytes()).await;
        }
    }
}
```

## 4. Plugin System (WASM Sandbox)

### 4.1 Why WASM instead of Deno

| | Deno | WASM (Wasmtime) |
|--|------|-----------------|
| Language | JavaScript/TypeScript | Rust (native), any lang via WASM |
| Sandbox | Process-level permissions | Memory-isolated, capability-based |
| Rust integration | FFI / subprocess | Native Wasmtime embedding |
| Startup time | ~50ms (V8 init) | ~1ms (pre-compiled module) |
| Dependency | Deno binary required | Embedded in ShopClaw binary |
| Plugin authoring | Need to write JS/TS | Write Rust, compile to wasm32-wasi |

### 4.2 Plugin SDK

```rust
// crates/shopclaw-plugin-sdk/src/lib.rs

//! SDK for writing ShopClaw plugins.
//! Plugins are compiled to `wasm32-wasi` and loaded by the ShopClaw host.

pub use shopclaw_plugin_sdk_macros::shopclaw_tool;

pub mod types;
pub mod bridge;
pub mod selector;

// Re-exports for convenience
pub use types::*;
pub use bridge::BrowserApi;
pub use selector::SelectorStore;
```

```rust
// crates/shopclaw-plugin-sdk/src/types.rs

use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Product {
    pub title: String,
    pub price: Price,
    pub url: String,
    pub image_url: Option<String>,
    pub rating: Option<f32>,
    pub review_count: Option<u32>,
    pub availability: Availability,
    pub seller: Option<String>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Price {
    pub amount: f64,
    pub currency: String,        // "USD", "CNY", "JPY"
    pub original: Option<f64>,   // original price before discount
    pub discount_pct: Option<f32>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(rename_all = "snake_case")]
pub enum Availability {
    InStock,
    LowStock,
    OutOfStock,
    PreOrder,
    Unknown,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct CartItem {
    pub product: Product,
    pub quantity: u32,
    pub subtotal: f64,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct CartSummary {
    pub items: Vec<CartItem>,
    pub total: f64,
    pub currency: String,
    pub item_count: u32,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct SearchResult {
    pub query: String,
    pub products: Vec<Product>,
    pub total_results: Option<u32>,
    pub page: u32,
    pub has_next_page: bool,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct PriceAlert {
    pub product_url: String,
    pub target_price: f64,
    pub current_price: f64,
    pub currency: String,
    pub triggered: bool,
}
```

```rust
// crates/shopclaw-plugin-sdk/src/bridge.rs

//! Host-imported browser operations.
//! These functions are provided by the ShopClaw host via WASM imports.
//! Plugin code calls these to interact with the browser.

/// Open a new tab with the given URL. Returns a tab ID.
#[link(wasm_import_module = "shopclaw")]
extern "C" {
    fn host_open_tab(url_ptr: *const u8, url_len: u32) -> u64;
    fn host_close_tab(tab_id: u64);
    fn host_navigate(tab_id: u64, url_ptr: *const u8, url_len: u32);
    fn host_wait_for_selector(tab_id: u64, sel_ptr: *const u8, sel_len: u32, timeout_ms: u32) -> u32;
    fn host_query_text(tab_id: u64, sel_ptr: *const u8, sel_len: u32, out_ptr: *mut u8, out_cap: u32) -> i32;
    fn host_click(tab_id: u64, sel_ptr: *const u8, sel_len: u32) -> u32;
    fn host_type_text(tab_id: u64, sel_ptr: *const u8, sel_len: u32, text_ptr: *const u8, text_len: u32) -> u32;
    fn host_get_url(tab_id: u64, out_ptr: *mut u8, out_cap: u32) -> i32;
    fn host_screenshot(tab_id: u64, out_ptr: *mut u8, out_cap: u32) -> i32;
    fn host_evaluate(tab_id: u64, expr_ptr: *const u8, expr_len: u32, out_ptr: *mut u8, out_cap: u32) -> i32;
}

/// Safe Rust wrapper around host imports.
pub struct BrowserApi;

impl BrowserApi {
    pub fn open_tab(url: &str) -> Result<u64, String> {
        let tab_id = unsafe { host_open_tab(url.as_ptr(), url.len() as u32) };
        if tab_id == 0 {
            Err("failed to open tab".into())
        } else {
            Ok(tab_id)
        }
    }

    pub fn click(tab_id: u64, selector: &str) -> Result<(), String> {
        let result = unsafe { host_click(tab_id, selector.as_ptr(), selector.len() as u32) };
        if result == 0 { Ok(()) } else { Err("click failed".into()) }
    }

    pub fn query_text(tab_id: u64, selector: &str) -> Result<String, String> {
        let mut buf = vec![0u8; 4096];
        let len = unsafe {
            host_query_text(tab_id, selector.as_ptr(), selector.len() as u32, buf.as_mut_ptr(), buf.len() as u32)
        };
        if len < 0 {
            Err("query_text failed".into())
        } else {
            buf.truncate(len as usize);
            String::from_utf8(buf).map_err(|e| e.to_string())
        }
    }

    pub fn wait_for(tab_id: u64, selector: &str, timeout_ms: u32) -> bool {
        let result = unsafe {
            host_wait_for_selector(tab_id, selector.as_ptr(), selector.len() as u32, timeout_ms)
        };
        result == 1
    }

    pub fn navigate(tab_id: u64, url: &str) {
        unsafe { host_navigate(tab_id, url.as_ptr(), url.len() as u32) };
    }

    pub fn close_tab(tab_id: u64) {
        unsafe { host_close_tab(tab_id) };
    }

    pub fn type_text(tab_id: u64, selector: &str, text: &str) -> Result<(), String> {
        let result = unsafe {
            host_type_text(
                tab_id,
                selector.as_ptr(), selector.len() as u32,
                text.as_ptr(), text.len() as u32,
            )
        };
        if result == 0 { Ok(()) } else { Err("type_text failed".into()) }
    }
}
```

### 4.3 Selector Store

```rust
// crates/shopclaw-plugin-sdk/src/selector.rs

use serde::{Deserialize, Serialize};
use std::collections::HashMap;

/// Manages CSS/XPath selectors for a shopping site.
/// Selectors can come from:
///   1. selectors.json (bundled with plugin, community-maintained)
///   2. selectors_cache.json (LLM-discovered, locally cached)
#[derive(Debug, Clone)]
pub struct SelectorStore {
    selectors: HashMap<String, SelectorEntry>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct SelectorEntry {
    /// Primary CSS selector
    pub css: String,
    /// Fallback XPath (optional)
    pub xpath: Option<String>,
    /// Human-readable description (for LLM fallback context)
    pub description: String,
    /// Last verified date (ISO 8601)
    pub last_verified: Option<String>,
}

impl SelectorStore {
    /// Load from a selectors.json string.
    pub fn from_json(json: &str) -> Result<Self, serde_json::Error> {
        let selectors: HashMap<String, SelectorEntry> = serde_json::from_str(json)?;
        Ok(Self { selectors })
    }

    /// Get the CSS selector for a named element.
    /// Returns None if not found (triggers LLM fallback in the host).
    pub fn get(&self, name: &str) -> Option<&str> {
        self.selectors.get(name).map(|e| e.css.as_str())
    }

    /// Get full entry including fallback xpath and description.
    pub fn get_entry(&self, name: &str) -> Option<&SelectorEntry> {
        self.selectors.get(name)
    }
}
```

### 4.4 Example Plugin: Amazon

```rust
// crates/shopclaw-plugin-amazon/src/lib.rs

use shopclaw_plugin_sdk::*;

mod search;
mod product;
mod cart;
mod price_track;

/// Plugin entry point — called by the host to register tools.
#[no_mangle]
pub extern "C" fn shopclaw_init() -> *const u8 {
    // Return a JSON manifest describing this plugin's tools.
    // The host parses this to register MCP tools.
    let manifest = include_str!("../manifest.json");
    manifest.as_ptr()
}

/// Tool dispatcher — called by the host when an Agent invokes a tool.
#[no_mangle]
pub extern "C" fn shopclaw_invoke(
    tool_ptr: *const u8,
    tool_len: u32,
    params_ptr: *const u8,
    params_len: u32,
    out_ptr: *mut u8,
    out_cap: u32,
) -> i32 {
    let tool_name = unsafe { std::str::from_utf8_unchecked(std::slice::from_raw_parts(tool_ptr, tool_len as usize)) };
    let params_str = unsafe { std::str::from_utf8_unchecked(std::slice::from_raw_parts(params_ptr, params_len as usize)) };
    let params: serde_json::Value = serde_json::from_str(params_str).unwrap_or_default();

    let result = match tool_name {
        "amazon_search" => search::execute(params),
        "amazon_product_detail" => product::detail(params),
        "amazon_add_to_cart" => cart::add(params),
        "amazon_view_cart" => cart::view(),
        "amazon_track_price" => price_track::track(params),
        _ => Err(format!("unknown tool: {}", tool_name)),
    };

    // Serialize result into output buffer
    let json = match result {
        Ok(v) => serde_json::to_string(&v).unwrap_or_default(),
        Err(e) => format!(r#"{{"error":"{}"}}"#, e),
    };

    let bytes = json.as_bytes();
    if bytes.len() > out_cap as usize {
        return -1; // buffer too small
    }
    unsafe {
        std::ptr::copy_nonoverlapping(bytes.as_ptr(), out_ptr, bytes.len());
    }
    bytes.len() as i32
}
```

```rust
// crates/shopclaw-plugin-amazon/src/search.rs

use shopclaw_plugin_sdk::*;

/// selectors.json for Amazon search results
const SELECTORS_JSON: &str = include_str!("../selectors.json");

pub fn execute(params: serde_json::Value) -> Result<serde_json::Value, String> {
    let query = params["query"].as_str().ok_or("missing 'query' parameter")?;
    let max_results = params["max_results"].as_u64().unwrap_or(10) as usize;
    let sort_by = params["sort_by"].as_str().unwrap_or("relevance");

    let selectors = SelectorStore::from_json(SELECTORS_JSON)
        .map_err(|e| format!("bad selectors.json: {}", e))?;

    // Build Amazon search URL
    let sort_param = match sort_by {
        "price_asc" => "&s=price-asc-rank",
        "price_desc" => "&s=price-desc-rank",
        "rating" => "&s=review-rank",
        _ => "",
    };
    let url = format!(
        "https://www.amazon.com/s?k={}{}",
        urlencoding::encode(query),
        sort_param
    );

    // Open tab and navigate
    let tab = BrowserApi::open_tab(&url)?;

    // Wait for search results to load
    let results_selector = selectors.get("search_result_item")
        .ok_or("selector 'search_result_item' not found")?;
    if !BrowserApi::wait_for(tab, results_selector, 10_000) {
        BrowserApi::close_tab(tab);
        return Err("search results did not load within 10s".into());
    }

    // Extract product data from each result
    let mut products = Vec::new();
    // ... iterate over result items, extract title/price/rating/url ...
    // (simplified — real implementation queries DOM for each element)

    let title_sel = selectors.get("result_title").unwrap_or("h2 a span");
    let price_sel = selectors.get("result_price").unwrap_or(".a-price .a-offscreen");
    let rating_sel = selectors.get("result_rating").unwrap_or(".a-icon-star-small .a-icon-alt");

    // For each result item, extract product info via BrowserApi::query_text
    // ... (omitted for brevity, but follows the same pattern)

    BrowserApi::close_tab(tab);

    let result = SearchResult {
        query: query.to_string(),
        products,
        total_results: None,
        page: 1,
        has_next_page: true,
    };

    serde_json::to_value(result).map_err(|e| e.to_string())
}
```

### 4.5 Example selectors.json (Amazon)

```json
{
  "search_result_item": {
    "css": "[data-component-type='s-search-result']",
    "description": "Each product card in search results"
  },
  "result_title": {
    "css": "h2 a span",
    "description": "Product title text within a search result"
  },
  "result_price": {
    "css": ".a-price .a-offscreen",
    "description": "Product price (screen-reader text for accurate parsing)"
  },
  "result_rating": {
    "css": ".a-icon-star-small .a-icon-alt",
    "description": "Rating text like '4.5 out of 5 stars'"
  },
  "result_url": {
    "css": "h2 a",
    "xpath": "//h2/a/@href",
    "description": "Product detail page URL"
  },
  "result_image": {
    "css": ".s-image",
    "description": "Product thumbnail image"
  },
  "add_to_cart_button": {
    "css": "#add-to-cart-button",
    "description": "Add to Cart button on product detail page"
  },
  "cart_count": {
    "css": "#nav-cart-count",
    "description": "Cart item count in navigation bar"
  },
  "cart_items": {
    "css": ".sc-list-item",
    "description": "Each item in the cart page"
  },
  "cart_subtotal": {
    "css": "#sc-subtotal-amount-activecart .sc-price",
    "description": "Cart subtotal price"
  },
  "checkout_button": {
    "css": "#sc-buy-box-ptc-button input",
    "description": "Proceed to checkout button on cart page"
  }
}
```

## 5. WASM Host Runtime

```rust
// crates/shopclaw-core/src/plugin/sandbox.rs

use wasmtime::*;
use std::path::Path;
use std::sync::Arc;
use tokio::sync::Mutex;
use crate::browser::BrowserBridge;

/// Loads and runs a plugin WASM module in a sandboxed environment.
pub struct PluginSandbox {
    engine: Engine,
    module: Module,
    store: Arc<Mutex<Store<HostState>>>,
    instance: Instance,
}

/// State accessible to the WASM plugin via host imports.
struct HostState {
    browser: Arc<dyn BrowserBridge>,
    plugin_name: String,
}

impl PluginSandbox {
    /// Load a plugin from a .wasm file.
    pub fn load(
        wasm_path: &Path,
        browser: Arc<dyn BrowserBridge>,
        plugin_name: &str,
    ) -> Result<Self> {
        let engine = Engine::default();
        let module = Module::from_file(&engine, wasm_path)?;

        let state = HostState {
            browser,
            plugin_name: plugin_name.to_string(),
        };
        let mut store = Store::new(&engine, state);

        let mut linker = Linker::new(&engine);

        // Register host-provided browser APIs as WASM imports.
        // Each import corresponds to a function in bridge.rs.
        linker.func_wrap("shopclaw", "host_open_tab", |caller: Caller<'_, HostState>, url_ptr: u32, url_len: u32| -> u64 {
            // Read URL from WASM memory, call browser.open_tab()
            // Return tab ID or 0 on failure
            todo!()
        })?;

        linker.func_wrap("shopclaw", "host_click", |caller: Caller<'_, HostState>, tab_id: u64, sel_ptr: u32, sel_len: u32| -> u32 {
            todo!()
        })?;

        // ... register remaining host functions ...

        let instance = linker.instantiate(&mut store, &module)?;

        Ok(Self {
            engine,
            module,
            store: Arc::new(Mutex::new(store)),
            instance,
        })
    }

    /// Call the plugin's shopclaw_init to get the manifest JSON.
    pub async fn get_manifest(&self) -> Result<String> {
        let mut store = self.store.lock().await;
        let init = self.instance
            .get_typed_func::<(), u32>(&mut *store, "shopclaw_init")?;
        let ptr = init.call(&mut *store, ())?;
        // Read manifest JSON string from WASM memory at ptr
        todo!()
    }

    /// Invoke a tool in the plugin.
    pub async fn invoke(
        &self,
        tool_name: &str,
        params: &serde_json::Value,
    ) -> Result<serde_json::Value> {
        let mut store = self.store.lock().await;
        let invoke_fn = self.instance
            .get_typed_func::<(u32, u32, u32, u32, u32, u32), i32>(
                &mut *store, "shopclaw_invoke"
            )?;
        // Write tool_name and params into WASM memory
        // Call invoke_fn, read result from output buffer
        todo!()
    }
}
```

## 6. MCP Server

```rust
// crates/shopclaw-core/src/mcp/server.rs

use crate::mcp::types::*;
use crate::mcp::router::Router;
use tokio::io::{self, AsyncBufReadExt, AsyncWriteExt, BufReader};

/// MCP server that communicates with the AI Agent over stdio.
pub struct McpServer {
    router: Router,
}

impl McpServer {
    pub fn new(router: Router) -> Self {
        Self { router }
    }

    /// Run the stdio JSON-RPC 2.0 event loop.
    pub async fn run(&self) -> Result<(), Box<dyn std::error::Error>> {
        let stdin = BufReader::new(io::stdin());
        let mut stdout = io::stdout();
        let mut lines = stdin.lines();

        while let Ok(Some(line)) = lines.next_line().await {
            let request: JsonRpcRequest = match serde_json::from_str(&line) {
                Ok(req) => req,
                Err(e) => {
                    let err_resp = JsonRpcResponse {
                        jsonrpc: "2.0".into(),
                        id: None,
                        result: None,
                        error: Some(JsonRpcError {
                            code: -32700,
                            message: format!("parse error: {}", e),
                            data: None,
                        }),
                    };
                    let json = serde_json::to_string(&err_resp)?;
                    stdout.write_all(format!("{}\n", json).as_bytes()).await?;
                    stdout.flush().await?;
                    continue;
                }
            };

            let response = self.handle_request(request).await;
            let json = serde_json::to_string(&response)?;
            stdout.write_all(format!("{}\n", json).as_bytes()).await?;
            stdout.flush().await?;
        }

        Ok(())
    }

    async fn handle_request(&self, request: JsonRpcRequest) -> JsonRpcResponse {
        match request.method.as_str() {
            // MCP initialization
            "initialize" => {
                JsonRpcResponse {
                    jsonrpc: "2.0".into(),
                    id: request.id,
                    result: Some(serde_json::json!({
                        "protocolVersion": "2024-11-05",
                        "capabilities": {
                            "tools": {}
                        },
                        "serverInfo": {
                            "name": "shopclaw",
                            "version": env!("CARGO_PKG_VERSION")
                        }
                    })),
                    error: None,
                }
            }

            // List available tools (from all loaded plugins)
            "tools/list" => {
                let tools = self.router.list_tools();
                JsonRpcResponse {
                    jsonrpc: "2.0".into(),
                    id: request.id,
                    result: Some(serde_json::json!({ "tools": tools })),
                    error: None,
                }
            }

            // Invoke a tool
            "tools/call" => {
                let params = request.params.unwrap_or_default();
                let tool_name = params["name"].as_str().unwrap_or("");
                let arguments = params.get("arguments").cloned().unwrap_or_default();

                match self.router.invoke(tool_name, arguments).await {
                    Ok(result) => JsonRpcResponse {
                        jsonrpc: "2.0".into(),
                        id: request.id,
                        result: Some(serde_json::json!({
                            "content": [{
                                "type": "text",
                                "text": serde_json::to_string(&result).unwrap_or_default()
                            }]
                        })),
                        error: None,
                    },
                    Err(e) => JsonRpcResponse {
                        jsonrpc: "2.0".into(),
                        id: request.id,
                        result: None,
                        error: Some(JsonRpcError {
                            code: -32000,
                            message: e.to_string(),
                            data: None,
                        }),
                    },
                }
            }

            _ => {
                JsonRpcResponse {
                    jsonrpc: "2.0".into(),
                    id: request.id,
                    result: None,
                    error: Some(JsonRpcError {
                        code: -32601,
                        message: format!("method not found: {}", request.method),
                        data: None,
                    }),
                }
            }
        }
    }
}
```

```rust
// crates/shopclaw-core/src/mcp/router.rs

use crate::gate::ConfirmationGate;
use crate::limiter::RateLimiter;
use crate::audit::{AuditLogger, AuditEntry, AuditResult};
use crate::plugin::sandbox::PluginSandbox;
use crate::mcp::types::*;
use std::collections::HashMap;
use std::sync::Arc;

/// Routes MCP tool calls to the correct plugin,
/// enforcing rate limits, confirmation gates, and audit logging.
pub struct Router {
    /// plugin_name → sandbox
    plugins: HashMap<String, Arc<PluginSandbox>>,
    /// tool_name → (plugin_name, ToolDefinition)
    tool_index: HashMap<String, (String, ToolDefinition)>,
    gate: ConfirmationGate,
    limiter: RateLimiter,
    audit: AuditLogger,
}

impl Router {
    /// List all tools from all loaded plugins (for MCP tools/list).
    pub fn list_tools(&self) -> Vec<serde_json::Value> {
        self.tool_index.values().map(|(_, def)| {
            serde_json::json!({
                "name": def.name,
                "description": def.description,
                "inputSchema": def.parameters,
            })
        }).collect()
    }

    /// Invoke a tool: gate check → rate limit → plugin invoke → audit log.
    pub async fn invoke(
        &self,
        tool_name: &str,
        params: serde_json::Value,
    ) -> Result<ToolResult, Box<dyn std::error::Error>> {
        // 1. Find the tool
        let (plugin_name, tool_def) = self.tool_index
            .get(tool_name)
            .ok_or_else(|| format!("tool not found: {}", tool_name))?;

        // 2. Confirmation gate
        if let Err(e) = self.gate.check(tool_name, tool_def.risk_level, &tool_def.description).await {
            self.audit.log(&AuditEntry {
                timestamp: chrono::Utc::now(),
                plugin: plugin_name.clone(),
                tool: tool_name.to_string(),
                risk_level: format!("{:?}", tool_def.risk_level),
                target_url: None,
                action: "invoke".into(),
                result: match &e {
                    crate::gate::GateError::Blocked(_) => AuditResult::Blocked,
                    crate::gate::GateError::UserDenied(_) => AuditResult::UserDenied,
                },
            }).await;
            return Err(e.into());
        }

        // 3. Rate limit
        let op_type = classify_operation(tool_name);
        if let Err(retry_after) = self.limiter.check(plugin_name, &op_type).await {
            self.audit.log(&AuditEntry {
                timestamp: chrono::Utc::now(),
                plugin: plugin_name.clone(),
                tool: tool_name.to_string(),
                risk_level: format!("{:?}", tool_def.risk_level),
                target_url: None,
                action: "invoke".into(),
                result: AuditResult::RateLimited,
            }).await;
            return Ok(ToolResult {
                success: false,
                data: serde_json::json!(null),
                error: Some(format!("rate limited, retry after {}s", retry_after.as_secs())),
                needs_user_action: false,
                user_action_prompt: None,
            });
        }

        // 4. Invoke plugin
        let sandbox = self.plugins.get(plugin_name)
            .ok_or_else(|| format!("plugin not loaded: {}", plugin_name))?;
        let result = sandbox.invoke(tool_name, &params).await?;

        // 5. Audit log
        self.audit.log(&AuditEntry {
            timestamp: chrono::Utc::now(),
            plugin: plugin_name.clone(),
            tool: tool_name.to_string(),
            risk_level: format!("{:?}", tool_def.risk_level),
            target_url: None,
            action: "invoke".into(),
            result: if result.get("error").is_some() {
                AuditResult::Failed(result["error"].as_str().unwrap_or("").to_string())
            } else {
                AuditResult::Success
            },
        }).await;

        Ok(ToolResult {
            success: result.get("error").is_none(),
            data: result,
            error: None,
            needs_user_action: false,
            user_action_prompt: None,
        })
    }
}

/// Classify a tool name into an operation type for rate limiting.
fn classify_operation(tool_name: &str) -> String {
    if tool_name.contains("search") { "search".into() }
    else if tool_name.contains("cart") { "cart".into() }
    else if tool_name.contains("track") { "price_track".into() }
    else { "navigate".into() }
}
```

## 7. CLI Interface

```rust
// crates/shopclaw-core/src/main.rs

use clap::{Parser, Subcommand};

#[derive(Parser)]
#[command(name = "shopclaw", version, about = "Local AI shopping assistant")]
struct Cli {
    #[command(subcommand)]
    command: Commands,
}

#[derive(Subcommand)]
enum Commands {
    /// Start the MCP server (stdio mode, for AI Agents)
    Serve {
        /// Config file path
        #[arg(short, long, default_value = "~/.shopclaw/config.toml")]
        config: String,
    },

    /// First-time setup: check extension, create config
    Setup,

    /// List installed plugins and their tools
    Plugins {
        #[command(subcommand)]
        action: PluginAction,
    },

    /// View or search the audit log
    Audit {
        /// Number of recent entries to show
        #[arg(short = 'n', long, default_value = "20")]
        last: usize,

        /// Filter by plugin name
        #[arg(short, long)]
        plugin: Option<String>,

        /// Filter by tool name
        #[arg(short, long)]
        tool: Option<String>,
    },
}

#[derive(Subcommand)]
enum PluginAction {
    /// List all installed plugins
    List,
    /// Install a plugin from a .wasm file or registry
    Install {
        /// Path to .wasm file or registry name (e.g., "amazon", "jd")
        source: String,
    },
    /// Uninstall a plugin
    Remove {
        /// Plugin name
        name: String,
    },
    /// Update selectors for a plugin
    UpdateSelectors {
        /// Plugin name
        name: String,
    },
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let cli = Cli::parse();

    match cli.command {
        Commands::Serve { config } => {
            // Load config, load plugins, start MCP server
            let config = load_config(&config)?;
            let browser = connect_browser().await?;
            let plugins = load_plugins(&config, browser.clone()).await?;
            let router = build_router(plugins, &config);
            let server = McpServer::new(router);
            server.run().await?;
        }
        Commands::Setup => {
            setup_wizard().await?;
        }
        Commands::Plugins { action } => {
            match action {
                PluginAction::List => list_plugins()?,
                PluginAction::Install { source } => install_plugin(&source).await?,
                PluginAction::Remove { name } => remove_plugin(&name)?,
                PluginAction::UpdateSelectors { name } => update_selectors(&name).await?,
            }
        }
        Commands::Audit { last, plugin, tool } => {
            show_audit_log(last, plugin, tool)?;
        }
    }

    Ok(())
}
```

## 8. Configuration

```toml
# ~/.shopclaw/config.toml

[general]
# Directory for plugin .wasm files
plugin_dir = "~/.shopclaw/plugins"
# Audit log path
audit_log = "~/.shopclaw/audit.log"

[browser]
# Browser Relay token file path
relay_token_path = "~/.shopclaw/relay-token"
# Browser Relay port file path
relay_port_path = "~/.shopclaw/relay-port"
# Maximum tabs to open simultaneously
max_concurrent_tabs = 3
# Default navigation timeout (ms)
navigation_timeout_ms = 15000

[rate_limit]
search_per_min = 10
navigate_per_min = 20
cart_per_min = 5
price_track_per_hour = 1

[confirmation]
# "native_dialog" or "terminal"
mode = "native_dialog"
# Write operations also require confirmation (default: false)
confirm_writes = false

[plugins.amazon]
enabled = true
regions = ["us", "jp"]

[plugins.jd]
enabled = true

[plugins.taobao]
enabled = false
```

## 9. Chrome Extension (MV3)

```
extension/manifest.json:
{
  "manifest_version": 3,
  "name": "ShopClaw Browser Relay",
  "version": "0.1.0",
  "description": "Connects ShopClaw to your browser for local AI shopping assistance",
  "permissions": ["debugger", "tabs", "storage"],
  "background": {
    "service_worker": "background.js"
  },
  "action": {
    "default_popup": "popup.html",
    "default_icon": "icon-48.png"
  },
  "icons": {
    "48": "icon-48.png",
    "128": "icon-128.png"
  }
}
```

The extension's service worker:
1. Generates a random token on first install, writes it to a well-known file via `chrome.downloads` API or native messaging
2. Starts a WebSocket server on a random localhost port, writes port to file
3. Validates token on incoming connections
4. Shows confirmation popup on first connection
5. Relays CDP commands via `chrome.debugger.sendCommand()`
6. Shows connection status in the extension popup (connected/disconnected, current operations)

Token and port file exchange between extension and ShopClaw uses **Native Messaging** (Chrome's official mechanism for extensions to communicate with local applications). The extension declares a native messaging host, and ShopClaw registers itself as a native messaging host during `shopclaw setup`.

## 10. Data Sanitization

```rust
// crates/shopclaw-core/src/mcp/router.rs (addition)

/// Whitelist filter applied to all plugin responses before returning to Agent.
fn sanitize_response(data: &serde_json::Value) -> serde_json::Value {
    // Deep-clone and strip sensitive fields
    let mut clean = data.clone();
    strip_fields(&mut clean, &[
        "password", "passwd", "secret",
        "cookie", "session_token", "csrf_token",
        "credit_card", "card_number", "cvv", "expiry",
        "full_address", "street", "postal_code", "zip",
        "phone", "email",  // only strip if not explicitly needed
        "ssn", "tax_id",
    ]);
    clean
}

fn strip_fields(value: &mut serde_json::Value, fields: &[&str]) {
    match value {
        serde_json::Value::Object(map) => {
            for field in fields {
                map.remove(*field);
            }
            for (_, v) in map.iter_mut() {
                strip_fields(v, fields);
            }
        }
        serde_json::Value::Array(arr) => {
            for v in arr.iter_mut() {
                strip_fields(v, fields);
            }
        }
        _ => {}
    }
}
```

## 11. LLM Fallback for Selector Discovery

When a selector in `selectors.json` fails (element not found), the host:

1. Takes a screenshot of the tab via `BrowserApi::screenshot()`
2. Gets the page HTML via `BrowserApi::get_html()`
3. Sends both to a local or remote LLM with the prompt:

```
I need to find the CSS selector for: {element_description}
on the page: {current_url}

Here is the page HTML (truncated to relevant section):
{html_snippet}

[Screenshot attached]

Return ONLY a JSON object: {"css": "...", "xpath": "..."}
```

4. Caches the discovered selector in `~/.shopclaw/selectors_cache/{plugin}/{element_name}.json`
5. Uses the cached selector for subsequent operations

The LLM integration is configurable:

```toml
# ~/.shopclaw/config.toml

[llm]
# "local" (Ollama), "anthropic", "openai", or "disabled"
provider = "anthropic"
model = "claude-sonnet-4-20250514"
# API key (read from env var ANTHROPIC_API_KEY if not set)
api_key_env = "ANTHROPIC_API_KEY"
# For local LLM
# provider = "local"
# endpoint = "http://localhost:11434"
# model = "llava"
```

## 12. Error Handling Strategy

```
Plugin error
  │
  ├─ SelectorNotFound → Trigger LLM fallback
  │    ├─ LLM finds selector → cache and retry
  │    └─ LLM fails → return needs_user_action: true
  │
  ├─ BrowserError::NotConnected → return error "extension not connected, run shopclaw setup"
  │
  ├─ BrowserError::UserDenied → return error "user denied operation"
  │
  ├─ RateLimited → return error with retry_after_secs
  │
  ├─ UserActionRequired (captcha, login page, etc.)
  │    → return needs_user_action: true + user_action_prompt
  │    → Agent shows prompt to user, waits, retries
  │
  └─ Other errors → return error message, log to audit
```

## 13. Testing Strategy

| Level | Tool | Scope |
|-------|------|-------|
| Unit tests | `cargo test` | Core logic: rate limiter, gate, sanitization, selector parsing |
| Plugin tests | `cargo test -p shopclaw-plugin-amazon` | Plugin logic against mock HTML fixtures |
| Integration tests | Headless Chrome + test HTTP server | Full flow: MCP call → plugin → browser → mock site |
| Extension tests | Playwright | Extension WebSocket relay, token exchange |

Mock test fixtures serve static HTML pages that mimic shopping site structures, allowing deterministic testing without hitting real sites.

## 14. Dependencies

```toml
# Cargo.toml (workspace)

[workspace]
members = [
    "crates/shopclaw-core",
    "crates/shopclaw-plugin-sdk",
    "crates/shopclaw-plugin-amazon",
    "crates/shopclaw-plugin-jd",
    "crates/shopclaw-plugin-taobao",
]

# crates/shopclaw-core/Cargo.toml
[dependencies]
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
clap = { version = "4", features = ["derive"] }
wasmtime = "19"
tokio-tungstenite = "0.23"
chrono = { version = "0.4", features = ["serde"] }
thiserror = "1"
anyhow = "1"
dirs = "5"
tracing = "0.1"
tracing-subscriber = "0.3"
native-dialog = "0.7"
toml = "0.8"

# crates/shopclaw-plugin-sdk/Cargo.toml
[lib]
crate-type = ["cdylib"]  # compile to .wasm

[dependencies]
serde = { version = "1", features = ["derive"] }
serde_json = "1"
urlencoding = "2"
```
