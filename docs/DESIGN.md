# ShopClaw — Technical Design Document

## 1. Project Overview

ShopClaw is a **local-first AI shopping assistant** that exposes shopping site operations as MCP (Model Context Protocol) tools. AI Agents discover and invoke these tools to search products, compare prices, and manage carts — all within the user's own browser.

**Language**: Rust (core + plugins)

**Browser Extension**: ShopClaw reuses the KleePay Browser Relay extension (repo: [Edmonds-LR/kleepay-browser-relay](https://github.com/Edmonds-LR/kleepay-browser-relay)), the same extension used by KleePay Signer.

**Key constraints**:
- No cloud backend, no telemetry, no credential storage
- No anti-bot circumvention; graceful degradation on captcha/risk-control
- No automatic payment; sensitive operations require user confirmation
- Plugin sandboxing via WebAssembly (Wasmtime)

## 2. Architecture Diagrams

### 2.1 全局架构总览

```
┌──────────────────────────────────────────────────────────────────────┐
│                                                                      │
│   用户："帮我比较一下 Amazon 和京东上 AirPods 的价格"                    │
│                                                                      │
│   ┌───────────────────────────────────────────────────────────┐      │
│   │  AI Agent (OpenClaw / Claude Code / Cursor)               │      │
│   │                                                           │      │
│   │  Agent 理解意图 → 决定调用 amazon_search + jd_search      │      │
│   └──────────────────────┬────────────────────────────────────┘      │
│                          │                                           │
│                          │ stdio (JSON-RPC 2.0)                      │
│                          │ MCP 协议                                   │
│                          ▼                                           │
│   ┌───────────────────────────────────────────────────────────┐      │
│   │  ShopClaw (Rust binary)                                   │      │
│   │                                                           │      │
│   │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌────────────┐ │      │
│   │  │   MCP   │  │ Plugin  │  │Selector │  │  Browser   │ │      │
│   │  │ Server  │→ │ Router  │→ │Resolver │→ │  Bridge    │ │      │
│   │  └─────────┘  └────┬────┘  └─────────┘  └──────┬─────┘ │      │
│   │                     │                            │       │      │
│   │              ┌──────┴──────┐                     │       │      │
│   │              ▼             ▼                     │       │      │
│   │        ┌──────────┐ ┌──────────┐                │       │      │
│   │        │  Amazon  │ │    JD    │  (WASM 沙箱)   │       │      │
│   │        │  Plugin  │ │  Plugin  │                │       │      │
│   │        └──────────┘ └──────────┘                │       │      │
│   └──────────────────────────────────────────────────┼───────┘      │
│                                                      │              │
│                                   WebSocket 127.0.0.1│              │
│                                                      ▼              │
│   ┌───────────────────────────────────────────────────────────┐     │
│   │  用户日常 Chrome                                           │     │
│   │                                                           │     │
│   │  ┌────────────────┐                                       │     │
│   │  │ KleePay Browser  │  chrome.debugger API                  │     │
│   │  │ Relay 扩展      │                                      │     │
│   │  └───────┬────────┘                                       │     │
│   │          │                                                │     │
│   │  ┌──────┴──────┐ ┌────────────┐ ┌────────────┐          │     │
│   │  │ Amazon Tab  │ │  JD Tab    │ │ Gmail Tab  │          │     │
│   │  │ (操作中)    │ │  (操作中)   │ │ (不受影响) │          │     │
│   │  └─────────────┘ └────────────┘ └────────────┘          │     │
│   └───────────────────────────────────────────────────────────┘     │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

### 2.2 一次 Tool 调用的完整路径

以 `amazon_search("AirPods")` 为例：

```
Agent                ShopClaw Core              Plugin (WASM)         KleePay 扩展         Amazon Tab
  │                      │                          │                     │                   │
  │ tools/call           │                          │                     │                   │
  │ "amazon_search"      │                          │                     │                   │
  │─────────────────────▶│                          │                     │                   │
  │                      │                          │                     │                   │
  │                      │ ① Confirmation Gate      │                     │                   │
  │                      │   risk=read → 放行       │                     │                   │
  │                      │                          │                     │                   │
  │                      │ ② Rate Limiter           │                     │                   │
  │                      │   search: 3/10 → 放行    │                     │                   │
  │                      │                          │                     │                   │
  │                      │ ③ invoke WASM            │                     │                   │
  │                      │─────────────────────────▶│                     │                   │
  │                      │                          │                     │                   │
  │                      │                          │ ④ selectors::get    │                   │
  │                      │      Selector Resolver ◀─│  ("search_result") │                   │
  │                      │      本地缓存命中 ──────▶│  → "[data-comp..]" │                   │
  │                      │                          │                     │                   │
  │                      │                          │ ⑤ browser::open_tab│                   │
  │                      │   Browser Bridge ◀───────│  (amazon.com/s?k=) │                   │
  │                      │                          │                     │                   │
  │                      │          WebSocket CDP ──┼─────────────────────▶ navigate          │
  │                      │                          │                     │──────────────────▶│
  │                      │                          │                     │   页面加载完成     │
  │                      │                          │                     │◀──────────────────│
  │                      │   tab_id ───────────────▶│                     │                   │
  │                      │                          │                     │                   │
  │                      │                          │ ⑥ browser::query_  │                   │
  │                      │   Browser Bridge ◀───────│   text(selector)   │                   │
  │                      │          CDP ────────────┼─────────────────────▶ DOM query         │
  │                      │                          │                     │◀─ "AirPods Pro.." │
  │                      │   text ─────────────────▶│                     │                   │
  │                      │                          │                     │                   │
  │                      │                          │ ⑦ 组装 SearchResult│                   │
  │                      │◀─────────────────────────│   返回 JSON        │                   │
  │                      │                          │                     │                   │
  │                      │ ⑧ Data Sanitize          │                     │                   │
  │                      │   (强类型，无敏感字段)     │                     │                   │
  │                      │                          │                     │                   │
  │                      │ ⑨ Audit Log              │                     │                   │
  │                      │   写入 audit.log          │                     │                   │
  │                      │                          │                     │                   │
  │ ⑩ 返回结果           │                          │                     │                   │
  │◀─────────────────────│                          │                     │                   │
  │  [{title: "AirPods   │                          │                     │                   │
  │    Pro", price: 249}]│                          │                     │                   │
```

### 2.3 Core 内部模块关系

```
┌─────────────────────────────────────────────────────────────┐
│  shopclaw-core                                              │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  main.rs (CLI)                                       │  │
│  │  shopclaw serve | setup | plugins | audit            │  │
│  └──────────────────────┬───────────────────────────────┘  │
│                         │                                   │
│  ┌──────────────────────▼───────────────────────────────┐  │
│  │  mcp/server.rs                                       │  │
│  │  stdio JSON-RPC 2.0 event loop                       │  │
│  │  ├── initialize → 返回 capabilities                  │  │
│  │  ├── tools/list → 汇总所有 Plugin 的 tools            │  │
│  │  └── tools/call → 转发给 Router                      │  │
│  └──────────────────────┬───────────────────────────────┘  │
│                         │                                   │
│  ┌──────────────────────▼───────────────────────────────┐  │
│  │  mcp/router.rs                                       │  │
│  │  ┌──────┐   ┌───────┐   ┌────────┐   ┌──────────┐  │  │
│  │  │ Gate │ → │Limiter│ → │ Plugin │ → │Sanitize  │  │  │
│  │  │ 风控 │   │ 限流  │   │ invoke │   │ + Audit  │  │  │
│  │  └──────┘   └───────┘   └────┬───┘   └──────────┘  │  │
│  └──────────────────────────────┼───────────────────────┘  │
│                                 │                           │
│  ┌──────────────────────────────▼───────────────────────┐  │
│  │  plugin/                                             │  │
│  │  ┌────────────┐  ┌─────────────┐  ┌──────────────┐ │  │
│  │  │ registry   │  │  sandbox    │  │  manifest    │ │  │
│  │  │ 热加载管理  │  │  Wasmtime   │  │  解析        │ │  │
│  │  │ 文件监听    │  │  WIT接口    │  │              │ │  │
│  │  └────────────┘  └─────────────┘  └──────────────┘ │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                             │
│  ┌──────────────────────┐  ┌────────────────────────────┐  │
│  │  selector/            │  │  browser/                  │  │
│  │  ┌────────────────┐  │  │  ┌──────────────────────┐  │  │
│  │  │ resolver       │  │  │  │ relay.rs             │  │  │
│  │  │ 三层解析       │  │  │  │ WebSocket → KleePay  │  │  │
│  │  │ Browser Relay 扩展   │  │  │
│  │  │ 缓存/远程/LLM │  │  │  └──────────────────────┘  │  │
│  │  ├────────────────┤  │  │  ┌──────────────────────┐  │  │
│  │  │ sync.rs        │  │  │  │ cdp.rs               │  │  │
│  │  │ 后台增量同步    │  │  │  │ CDP 命令构建          │  │  │
│  │  ├────────────────┤  │  │  └──────────────────────┘  │  │
│  │  │ health.rs      │  │  │  ┌──────────────────────┐  │  │
│  │  │ 选择器健康检查  │  │  │  │ tab.rs               │  │  │
│  │  └────────────────┘  │  │  │ Tab 生命周期管理       │  │  │
│  └──────────────────────┘  │  └──────────────────────┘  │  │
│                             └────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### 2.4 Selector 三层解析流程

```
Plugin 请求: selectors::get("search_result_item")
  │
  ▼
┌─────────────────────────────────────────────────────────────┐
│  Layer 1: 本地缓存                                          │
│  ~/.shopclaw/selectors/amazon/selectors.json                │
│                                                             │
│  查找 "search_result_item" ──► 找到                         │
│  │                                                          │
│  ▼                                                          │
│  在当前页面验证 selector → 能匹配到元素？                     │
│  ├── Yes ──► 返回 ✓                                        │
│  └── No ───► selector 过期，降级 ↓                          │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│  Layer 2: 从 Repo 拉取最新                                   │
│                                                             │
│  GET raw.githubusercontent.com/tokligence/ShopClaw/         │
│      main/registry/amazon/selectors.json                    │
│                                                             │
│  ├── 有新版本 → 更新本地缓存 → 验证 → 匹配？               │
│  │   ├── Yes ──► 返回 ✓                                    │
│  │   └── No ───► 降级 ↓                                    │
│  │                                                          │
│  └── 网络不可达 / 无更新 ──► 降级 ↓                          │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│  Layer 3: LLM 实时发现                                       │
│                                                             │
│  截图当前页面 + 获取 HTML                                    │
│       │                                                     │
│       ▼                                                     │
│  发送给 LLM:                                                │
│  "在 amazon.com 搜索结果页找 '每个商品卡片' 的 CSS selector" │
│       │                                                     │
│       ▼                                                     │
│  LLM 返回: "[data-component-type='s-search-result']"        │
│       │                                                     │
│       ▼                                                     │
│  在页面验证 → 匹配？                                         │
│  ├── Yes ──► 写入 Layer 1 缓存 ──► 返回 ✓                  │
│  │           可选: 回报社区 (PR)                              │
│  └── No  ──► 返回错误，通知 Agent 需要人工介入               │
└─────────────────────────────────────────────────────────────┘
```

### 2.5 Repo 内 Registry 结构与同步

```
GitHub Repo: tokligence/ShopClaw
│
├── crates/                    # Rust 源码
│                              # Chrome extension: see Edmonds-LR/kleepay-browser-relay
├── DESIGN.md
│
└── registry/                  # ◄── Selector + Plugin 更新源
    ├── manifest.json          # 全局版本清单
    ├── amazon/
    │   ├── selectors.json     # Amazon 最新选择器
    │   ├── meta.json          # 版本、更新时间、兼容的 Plugin 版本
    │   └── plugin.wasm        # (可选) 预编译 Plugin binary
    ├── jd/
    │   ├── selectors.json
    │   ├── meta.json
    │   └── plugin.wasm
    └── taobao/
        ├── selectors.json
        ├── meta.json
        └── plugin.wasm


同步数据流:

┌────────────────────────────────┐
│  GitHub Repo                   │
│  registry/                     │
│  ├── manifest.json             │
│  ├── amazon/selectors.json     │ ◄── 社区 PR / 维护者更新
│  ├── jd/selectors.json         │
│  └── taobao/selectors.json     │
└───────────────┬────────────────┘
                │
                │ raw.githubusercontent.com
                │ (每 6h 检查一次)
                ▼
┌────────────────────────────────────────────────────────────┐
│  用户机器上的 ShopClaw                                       │
│                                                            │
│  ┌─────────────────────┐     ┌──────────────────────────┐ │
│  │ SelectorSyncService │     │ PluginUpdateChecker      │ │
│  │                     │     │                          │ │
│  │ ① 拉 manifest.json │     │ ① 拉 manifest.json      │ │
│  │ ② 比较版本号        │     │ ② 检查有无新 plugin.wasm │ │
│  │ ③ 有更新就下载      │     │ ③ 有更新就下载到         │ │
│  │   selectors.json    │     │   ~/.shopclaw/plugins/   │ │
│  │ ④ 写入本地缓存      │     │ ④ 文件监听器自动热加载   │ │
│  └─────────────────────┘     └──────────────────────────┘ │
│            │                              │                │
│            ▼                              ▼                │
│  ~/.shopclaw/                  ~/.shopclaw/plugins/        │
│  selectors/                    ├── amazon.wasm             │
│  ├── amazon/selectors.json     ├── jd.wasm                │
│  ├── jd/selectors.json         └── taobao.wasm            │
│  └── taobao/selectors.json                                │
└────────────────────────────────────────────────────────────┘
```

### 2.6 Plugin WASM 沙箱隔离

```
┌──────────────────────────────────────────────────────┐
│  ShopClaw 进程                                        │
│                                                      │
│  ┌────────────────────────────────────────────────┐  │
│  │  Wasmtime Runtime                              │  │
│  │                                                │  │
│  │  ┌──────────────┐   ┌──────────────┐          │  │
│  │  │ Amazon WASM  │   │   JD WASM    │  ...     │  │
│  │  │              │   │              │          │  │
│  │  │ 独立内存空间  │   │ 独立内存空间  │          │  │
│  │  │ 无文件系统    │   │ 无文件系统    │          │  │
│  │  │ 无网络访问    │   │ 无网络访问    │          │  │
│  │  │ 无法读其他    │   │ 无法读其他    │          │  │
│  │  │ Plugin 数据   │   │ Plugin 数据   │          │  │
│  │  │              │   │              │          │  │
│  │  │ 只能调用:     │   │ 只能调用:     │          │  │
│  │  │ · browser::* │   │ · browser::* │          │  │
│  │  │ · selectors::│   │ · selectors::│          │  │
│  │  └──────┬───────┘   └──────┬───────┘          │  │
│  │         │ WIT 接口         │ WIT 接口          │  │
│  └─────────┼──────────────────┼───────────────────┘  │
│            │                  │                       │
│            ▼                  ▼                       │
│  ┌────────────────────────────────────────────────┐  │
│  │  Host API (Rust native)                        │  │
│  │  Browser Bridge / Selector Resolver / Gate     │  │
│  └────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────┘

Plugin 能做的:                     Plugin 不能做的:
 ✓ 打开/关闭 Tab                    ✗ 读写文件系统
 ✓ 点击/输入/读取 DOM               ✗ 发起网络请求
 ✓ 请求选择器                       ✗ 访问其他 Plugin 数据
 ✓ 截图 (用于 LLM)                  ✗ 执行系统命令
                                    ✗ 访问用户凭据/Cookie
```

### 2.7 KleePay Browser Relay 扩展交互

```
ShopClaw Core                    Chrome Extension                User Chrome
     │                                │                              │
     │ ① 读 ~/.shopclaw/relay-token   │                              │
     │    读 ~/.shopclaw/relay-port    │                              │
     │                                │                              │
     │ ② ws://127.0.0.1:PORT         │                              │
     │    + token                     │                              │
     │───────────────────────────────▶│                              │
     │                                │ ③ 验证 token                 │
     │                                │                              │
     │                                │ ④ 弹窗确认 ─────────────────▶│
     │                                │   "ShopClaw 请求连接"        │ 用户点「允许」
     │                                │◀─────────────────────────────│
     │                                │                              │
     │◀── 信任会话建立 ───────────────│                              │
     │    (无超时，断开即失效)          │                              │
     │                                │                              │
     │ ⑤ CDP: Target.createTarget    │                              │
     │   url: "amazon.com/s?k=..."    │                              │
     │───────────────────────────────▶│                              │
     │                                │ chrome.debugger              │
     │                                │   .attach(tabId)             │
     │                                │──────────────────────────────▶ 新 Tab 打开
     │                                │                              │
     │◀── tabId ──────────────────────│                              │
     │                                │                              │
     │ ⑥ CDP: DOM.querySelector      │                              │
     │   selector: "[data-comp...]"   │                              │
     │───────────────────────────────▶│                              │
     │                                │ chrome.debugger              │
     │                                │   .sendCommand(              │
     │                                │     tabId,                   │
     │                                │     "DOM.querySelector",     │
     │                                │     {selector: "..."}        │
     │                                │   )                          │
     │                                │──────────────────────────────▶ DOM 查询
     │                                │◀───── nodeId ────────────────│
     │◀── nodeId ─────────────────────│                              │
     │                                │                              │
     │ ... 更多操作 ...               │  扩展状态栏:                  │
     │                                │  "🟢 ShopClaw 操作中:       │
     │                                │   amazon.com"                │
     │                                │                              │
     │ ⑦ 断开                        │                              │
     │───────────────────────────────▶│                              │
     │                                │  扩展状态栏:                  │
     │                                │  "⚪ 未连接"                 │
```

### 2.8 跨站比价完整流程

```
用户: "比较 Amazon 和京东上 AirPods 的价格"
  │
  ▼
Agent 规划: 需要并行调用两个 search tool
  │
  ├─────────────────────────┐
  │                         │
  ▼                         ▼
amazon_search("AirPods")   jd_search("AirPods")
  │                         │
  ▼                         ▼
ShopClaw Router            ShopClaw Router
  │                         │
  ▼                         ▼
Amazon Plugin (WASM)       JD Plugin (WASM)
  │                         │
  ▼                         ▼
打开 Amazon Tab            打开 JD Tab
提取商品数据               提取商品数据
关闭 Tab                   关闭 Tab
  │                         │
  ▼                         ▼
返回 SearchResult          返回 SearchResult
  │                         │
  └───────────┬─────────────┘
              │
              ▼
Agent 整合结果:

  "我找到了以下对比:

   | 商品         | Amazon    | 京东      |
   |-------------|-----------|-----------|
   | AirPods Pro | $249.00   | ¥1,799    |
   | AirPods 3   | $169.00   | ¥1,279    |

   Amazon 更便宜。要加入购物车吗？"
```

### 2.9 Selector 过期 → 自动恢复流程

```
用户: "搜索 Amazon 上的充电器"
  │
  ▼
Plugin 调用 selectors::get("search_result_item")
  │
  ▼
本地缓存返回: "[data-component-type='s-search-result']"
  │
  ▼
Browser Bridge 在页面上查找 → 匹配 0 个元素 ❌
  │
  ▼  (Amazon 上周改版了!)
  │
  ▼
ShopClaw 自动触发 Layer 2:
从 Repo 拉取最新 selectors.json
  │
  ├─ 有更新! 新 selector: ".s-result-item[data-asin]"
  │  在页面验证 → 匹配到 20 个元素 ✓
  │  更新本地缓存
  │  │
  │  ▼
  │  正常返回搜索结果给用户
  │  (用户完全无感知)
  │
  └─ 没更新 (社区还没来得及修)
     │
     ▼
     ShopClaw 自动触发 Layer 3:
     截图 + HTML → LLM
     │
     ▼
     LLM: "新的 selector 是 .s-main-slot .s-result-item"
     │
     ▼
     验证 → 匹配成功 ✓
     写入本地缓存
     │
     ▼
     正常返回搜索结果给用户
     (用户完全无感知)
```

## 3. Workspace Layout

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
│                                 # Chrome extension lives in separate repo:
│                                 # Edmonds-LR/kleepay-browser-relay
│
├── registry/                     # Selector + Plugin 更新源 (in-repo)
│   ├── manifest.json             # 全局版本清单
│   ├── amazon/
│   │   ├── selectors.json        # 最新 Amazon 选择器
│   │   ├── meta.json             # 版本号、更新时间
│   │   └── plugin.wasm           # (可选) 预编译 Plugin
│   ├── jd/
│   │   ├── selectors.json
│   │   ├── meta.json
│   │   └── plugin.wasm
│   └── taobao/
│       ├── selectors.json
│       ├── meta.json
│       └── plugin.wasm
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

/// Manages the WebSocket connection to the KleePay Browser Relay extension.
/// ShopClaw reads relay-token and relay-port files written by the KleePay extension.
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

    /// Discover the KleePay extension's WebSocket port.
    /// The KleePay Browser Relay extension writes its port to ~/.shopclaw/relay-port.
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

    /// Connect to the KleePay Browser Relay extension's WebSocket relay.
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

## 9. Chrome Extension (KleePay Browser Relay)

> **The Chrome extension is maintained in a separate repo: [Edmonds-LR/kleepay-browser-relay](https://github.com/Edmonds-LR/kleepay-browser-relay).**
>
> ShopClaw does not ship its own extension. It connects to the KleePay Browser Relay extension the same way KleePay Signer does — by reading the `relay-token` and `relay-port` files that the extension writes, then opening a WebSocket connection authenticated with the token.
>
> The extension provides CDP relay via `chrome.debugger.sendCommand()`, token-based auth, and a user-facing connection status popup. See the kleepay-browser-relay repo for extension internals (manifest, service worker, native messaging setup).

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

## 15. Design Review — 已识别问题与修正

### 15.1 问题 1：WASM FFI 太底层，Plugin 开发体验差

**现状**：Plugin 通过 `extern "C"` + 裸指针 (`*const u8`, `u32` len) 与宿主通信。开发者要手写 unsafe 代码处理内存布局。

**问题**：
- 容易写出内存安全 bug（越界、悬垂指针）
- 开发门槛高，社区贡献者不愿意写
- 每个 Plugin 重复大量样板代码

**修正**：使用 **WASM Component Model + wit-bindgen** 替代裸 FFI。

```wit
// shopclaw.wit — 定义宿主与插件的类型安全接口

package shopclaw:plugin@0.1.0;

interface browser {
    record tab {
        id: u64,
        url: string,
        title: string,
    }

    open-tab: func(url: string) -> result<tab, string>;
    close-tab: func(tab-id: u64);
    navigate: func(tab-id: u64, url: string) -> result<_, string>;
    wait-for-selector: func(tab-id: u64, selector: string, timeout-ms: u32) -> bool;
    query-text: func(tab-id: u64, selector: string) -> result<string, string>;
    click: func(tab-id: u64, selector: string) -> result<_, string>;
    type-text: func(tab-id: u64, selector: string, text: string) -> result<_, string>;
    get-url: func(tab-id: u64) -> result<string, string>;
    screenshot: func(tab-id: u64) -> result<list<u8>, string>;
    evaluate: func(tab-id: u64, expression: string) -> result<string, string>;
}

interface selectors {
    /// 宿主侧加载选择器（不是 Plugin 内 include_str!）
    get: func(name: string) -> option<string>;
    get-or-llm-fallback: func(name: string, description: string) -> result<string, string>;
}

world plugin {
    import browser;
    import selectors;

    export init: func() -> string;          // 返回 manifest JSON
    export invoke: func(tool: string, params: string) -> string;  // 返回 result JSON
}
```

**改后 Plugin 代码**（无 unsafe）：

```rust
// crates/shopclaw-plugin-amazon/src/search.rs (使用 wit-bindgen)

use crate::bindings::browser;
use crate::bindings::selectors;

pub fn execute(params: serde_json::Value) -> Result<serde_json::Value, String> {
    let query = params["query"].as_str().ok_or("missing 'query'")?;
    let url = format!("https://www.amazon.com/s?k={}", urlencoding::encode(query));

    // 类型安全，无 unsafe
    let tab = browser::open_tab(&url)?;
    let sel = selectors::get_or_llm_fallback(
        "search_result_item",
        "Each product card in Amazon search results"
    )?;

    if !browser::wait_for_selector(tab.id, &sel, 10_000) {
        browser::close_tab(tab.id);
        return Err("search results did not load".into());
    }

    let title = browser::query_text(tab.id,
        &selectors::get_or_llm_fallback("result_title", "Product title in search result")?
    )?;

    browser::close_tab(tab.id);
    // ...
    Ok(serde_json::json!({"title": title}))
}
```

### 15.2 问题 2：WASM 内异步操作无法工作

**现状**：Browser Bridge 是 `async fn`，但 WASM 不原生支持 async。Plugin 调 `host_open_tab()` 时，宿主需要做异步网络操作（WebSocket → KleePay Browser Relay 扩展 → CDP），但 WASM 调用是同步阻塞的。

**修正**：宿主在 host import 实现中使用 `block_on` 桥接。Wasmtime 的 host function 可以通过 `tokio::runtime::Handle::current().block_on()` 同步等待异步操作完成。Plugin 侧无需关心——它调用的是同步 API，阻塞等待结果返回。

```rust
// 宿主侧 host function 实现
linker.func_wrap("browser", "open-tab", |mut caller: Caller<'_, HostState>, url: &str| -> Result<Tab> {
    let browser = caller.data().browser.clone();
    let handle = tokio::runtime::Handle::current();

    // 同步阻塞等待异步操作完成
    let tab = handle.block_on(async {
        browser.open_tab(url).await
    })?;

    Ok(tab)
})?;
```

这是有意的设计取舍：Plugin 执行是串行阻塞的（一个 Plugin 同时只有一个 tool 在执行），但多个 Plugin 之间可以并行（每个 Plugin 有独立的 WASM 实例和 Store）。

### 15.3 问题 3：selectors.json 用 `include_str!` 编译进 WASM，更新需重编译

**现状**：`const SELECTORS_JSON: &str = include_str!("../selectors.json");` 把选择器编译进 WASM binary。Amazon 改版 → 选择器失效 → 需要重新编译 Plugin WASM → 用户需要重新下载安装。

**这是你提的热加载问题的根本原因。详见 §16 Selector 热更新系统。**

### 15.4 问题 4：`sanitize_response` 按字段名黑名单过滤太脆弱

**现状**：按 `"password"`, `"cookie"` 等字段名匹配然后删除。

**问题**：
- Plugin 可以用 `"passwd"`, `"pw"`, `"credentials"` 等变体名绕过
- 合法字段如 `"email"` 在某些场景（如订单确认）是需要的

**修正**：Plugin 通过 WIT 接口返回**强类型结构**（`Product`, `SearchResult`, `CartSummary`），而不是任意 JSON。宿主只序列化白名单类型中的字段，Plugin 无法返回任意数据。

```wit
// 在 shopclaw.wit 中定义返回类型
record product {
    title: string,
    price: f64,
    currency: string,
    url: string,
    rating: option<f32>,
    availability: string,
}

record search-result {
    query: string,
    products: list<product>,
    has-next-page: bool,
}

// Plugin 必须返回这些类型，无法塞入任意字段
```

### 15.5 问题 5：Plugin SDK 的 `crate-type = ["cdylib"]` 标注位置错误

**现状**：`shopclaw-plugin-sdk/Cargo.toml` 标注了 `crate-type = ["cdylib"]`。

**问题**：SDK 是一个库（被 Plugin 引用），不应该自己编译为 cdylib。应该是**各个 Plugin crate** 标注 `crate-type = ["cdylib"]`。

**修正**：
```toml
# crates/shopclaw-plugin-sdk/Cargo.toml
[lib]
crate-type = ["rlib"]  # 普通 Rust 库，被 Plugin 依赖

# crates/shopclaw-plugin-amazon/Cargo.toml
[lib]
crate-type = ["cdylib"]  # 编译为 .wasm
[dependencies]
shopclaw-plugin-sdk = { path = "../shopclaw-plugin-sdk" }
```

---

## 16. Selector 热更新系统

### 16.1 设计目标

购物网站经常改版（Amazon 平均每 2-4 周调整一次前端结构）。Selector 失效后，用户不应该需要重新安装 ShopClaw 或更新 Plugin。

**目标**：
1. Selector 与 Plugin WASM 解耦——WASM 管逻辑，Selector 在宿主侧独立管理
2. 自动检测 Selector 失效
3. 自动从远程拉取最新 Selector
4. LLM 兜底发现新 Selector 并回报社区

### 16.2 三层 Selector 解析

```
Plugin 请求 selector "search_result_item"
  │
  ▼
┌──────────────────────────────────────────────────────┐
│  宿主 SelectorResolver                                │
│                                                      │
│  Layer 1: 本地缓存                                    │
│  ~/.shopclaw/selectors/{plugin}/selectors_cache.json  │
│  ├─ 命中 → 返回                                      │
│  └─ 未命中 ↓                                          │
│                                                      │
│  Layer 2: 远程 Selector Registry                      │
│  GET https://raw.githubusercontent.com/tokligence/ShopClaw/main/registry/     │
│      {plugin}/{selector_name}                        │
│  ├─ 命中 → 写入 Layer 1 缓存 → 返回                   │
│  └─ 未命中 ↓                                          │
│                                                      │
│  Layer 3: LLM 实时发现                                │
│  截图 + HTML → LLM → 推断 CSS selector                │
│  ├─ 成功 → 写入 Layer 1 缓存 → 可选上报 Registry → 返回│
│  └─ 失败 → 返回 Err，Plugin 通知 Agent 需要人工介入    │
└──────────────────────────────────────────────────────┘
```

### 16.3 Selector Registry（远程选择器仓库）

Selector Registry 是一个轻量级 HTTP 服务，托管社区维护的最新选择器：

```
https://registry.shopclaw.dev/
  └── v1/
      └── selectors/
          ├── amazon/
          │   ├── manifest.json          # 版本号 + 最后更新时间
          │   └── selectors.json         # 所有 Amazon 选择器
          ├── jd/
          │   ├── manifest.json
          │   └── selectors.json
          └── taobao/
              ├── manifest.json
              └── selectors.json
```

**不需要自建服务器**——直接用 ShopClaw Repo 的 `registry/` 文件夹，通过 GitHub Raw 访问：

```
https://raw.githubusercontent.com/tokligence/ShopClaw/main/registry/amazon/selectors.json
```

代码和选择器在同一个 Repo 里，社区通过 PR 更新 `registry/` 目录下的 JSON，merge 后所有用户自动获取最新版本。

### 16.4 自动检测 Selector 失效

```rust
// crates/shopclaw-core/src/selector/health.rs

/// Selector 健康检查策略
pub struct SelectorHealthChecker;

impl SelectorHealthChecker {
    /// 在每次 Plugin 调用 `get_or_llm_fallback` 时，宿主先用 selector 探测页面。
    /// 如果 selector 匹配到 0 个元素 → 标记为 stale。
    pub async fn check(
        &self,
        browser: &dyn BrowserBridge,
        tab_id: &str,
        selector: &str,
    ) -> SelectorHealth {
        match browser.query_selector_all(tab_id, selector).await {
            Ok(elements) if elements.is_empty() => SelectorHealth::Stale,
            Ok(_) => SelectorHealth::Healthy,
            Err(_) => SelectorHealth::Stale,
        }
    }
}

pub enum SelectorHealth {
    Healthy,
    Stale,   // selector 可能过期，触发更新
}
```

**完整流程**：

```
Plugin 调用 selectors::get("search_result_item")
  │
  ▼
宿主从 Layer 1 缓存拿到 selector
  │
  ▼
宿主用 selector 在当前页面探测 → 匹配到元素？
  ├─ Yes → 返回 selector（健康）
  └─ No  → selector 可能过期
            │
            ▼
         从 Registry 拉取最新 selector
            │
            ├─ 新 selector ≠ 旧 selector → 用新的探测
            │    ├─ 匹配成功 → 更新缓存，返回新 selector
            │    └─ 匹配失败 → 降级 LLM
            │
            └─ Registry 无更新 / 网络不可达 → 降级 LLM
                  │
                  ▼
               LLM 截图分析
                  ├─ 发现新 selector → 缓存 + 可选上报 → 返回
                  └─ 失败 → 返回错误
```

### 16.5 后台增量同步

不需要每次调用都查 Registry。ShopClaw 启动时和运行中定期后台同步：

```rust
// crates/shopclaw-core/src/selector/sync.rs

use std::time::Duration;

pub struct SelectorSyncService {
    registry_base_url: String,
    cache_dir: PathBuf,
    sync_interval: Duration,
}

impl SelectorSyncService {
    /// 启动后台同步任务
    pub fn spawn(self) -> tokio::task::JoinHandle<()> {
        tokio::spawn(async move {
            loop {
                for plugin in &["amazon", "jd", "taobao"] {
                    if let Err(e) = self.sync_plugin(plugin).await {
                        tracing::warn!("selector sync failed for {}: {}", plugin, e);
                    }
                }
                tokio::time::sleep(self.sync_interval).await;
            }
        })
    }

    async fn sync_plugin(&self, plugin: &str) -> Result<()> {
        // 1. 获取远程 manifest（版本号 + etag）
        let url = format!("{}/{}/manifest.json", self.registry_base_url, plugin);
        let remote_manifest: SelectorManifest = reqwest::get(&url).await?.json().await?;

        // 2. 比较本地版本
        let local_manifest = self.load_local_manifest(plugin)?;
        if local_manifest.version >= remote_manifest.version {
            return Ok(()); // 已是最新
        }

        // 3. 下载新 selectors.json
        let selectors_url = format!("{}/{}/selectors.json", self.registry_base_url, plugin);
        let new_selectors = reqwest::get(&selectors_url).await?.text().await?;

        // 4. 写入本地缓存
        let cache_path = self.cache_dir.join(plugin).join("selectors.json");
        tokio::fs::create_dir_all(cache_path.parent().unwrap()).await?;
        tokio::fs::write(&cache_path, &new_selectors).await?;

        // 5. 更新本地 manifest
        let manifest_path = self.cache_dir.join(plugin).join("manifest.json");
        tokio::fs::write(&manifest_path, serde_json::to_string(&remote_manifest)?).await?;

        tracing::info!(
            "updated selectors for {} : v{} → v{}",
            plugin, local_manifest.version, remote_manifest.version
        );
        Ok(())
    }
}

#[derive(Debug, Serialize, Deserialize)]
struct SelectorManifest {
    version: u32,
    updated_at: String,  // ISO 8601
    plugin: String,
    selector_count: usize,
}
```

### 16.6 LLM 发现的 Selector 回报社区（可选）

用户可以 opt-in 把 LLM 发现的新 selector 自动提交给社区：

```toml
# ~/.shopclaw/config.toml

[selector_sync]
# 从 GitHub raw 拉取最新选择器
registry_url = "https://raw.githubusercontent.com/tokligence/ShopClaw/main/registry"
# 同步间隔
sync_interval = "6h"
# 是否把 LLM 发现的新 selector 回报社区（需要 GitHub token）
contribute_discoveries = true
# GitHub token（仅用于创建 PR，可选）
github_token_env = "GITHUB_TOKEN"
```

**回报流程**：

```
LLM 发现新 selector
  │
  ▼
本地验证：在当前页面用新 selector 能匹配到正确元素？
  ├─ No → 只存本地缓存，不上报
  └─ Yes ↓
        │
        ▼
    contribute_discoveries = true?
      ├─ No → 只存本地缓存
      └─ Yes ↓
            │
            ▼
        自动 fork tokligence/ShopClaw repo
        更新 registry/{plugin}/selectors.json
        创建 PR：
          "Auto-discovered selector: {plugin}/{name}"
          Body: "Discovered by LLM on {date}, verified on {url}"
        │
        ▼
    社区 maintainer review + merge
        │
        ▼
    所有用户在下次同步时自动获取
```

### 16.7 完整数据流

```
                         ┌─────────────────────────┐
                         │  GitHub Repo             │
                         │  tokligence/ShopClaw     │
                         │  registry/               │
                         │  ├── amazon/             │
                         │  │   └── selectors.json  │ ◄── 社区 PR 更新
                         │  ├── jd/                 │
                         │  └── taobao/             │
                         └───────────┬──────────────┘
                                     │ raw.githubusercontent.com
                                     │ (每 6h 同步一次)
                                     ▼
┌────────────────────────────────────────────────────────────┐
│  用户机器                                                    │
│                                                            │
│  ~/.shopclaw/selectors/              ShopClaw Core          │
│  ├── amazon/                         ┌────────────────┐    │
│  │   ├── selectors.json  ◄───────── │ SelectorSync   │    │
│  │   ├── manifest.json               │ Service        │    │
│  │   └── selectors_cache.json ◄──── │ (后台增量同步)  │    │
│  ├── jd/                             └────────────────┘    │
│  └── taobao/                                  ▲            │
│                                               │            │
│  Plugin (WASM) 请求 selector                   │            │
│       │                                       │            │
│       ▼                                       │            │
│  SelectorResolver                             │            │
│  ├─ Layer 1: 本地缓存 ─── hit ──► 返回       │            │
│  ├─ Layer 2: 远程同步 ─── hit ──► 缓存+返回  │            │
│  └─ Layer 3: LLM 发现 ─── hit ──► 缓存+返回  │            │
│                     │                         │            │
│                     └── opt-in 回报 ──────────┘            │
└────────────────────────────────────────────────────────────┘
```

### 16.8 用户视角

用户完全无感知。从用户角度看：

| 场景 | 用户需要做什么 |
|------|--------------|
| Amazon 改版，选择器失效 | **什么都不用做**。ShopClaw 后台自动拉取最新选择器 |
| 社区还没更新选择器 | **什么都不用做**。LLM 自动发现新选择器，本地缓存 |
| LLM 也找不到 | ShopClaw 提示「Amazon 页面结构变化较大，请稍后重试」|
| ShopClaw 本身有 bug fix | 需要更新 ShopClaw binary（`cargo install --force` 或包管理器）|

**关键区分**：
- **Selector 更新** = 不需要重装软件，自动同步
- **Plugin 逻辑更新**（如 Amazon 改了 checkout 流程）= 需要更新 Plugin WASM，但也可以热加载（放入 `~/.shopclaw/plugins/` 目录即可，无需重启）
- **Core 更新** = 需要重装 ShopClaw binary

### 16.9 Plugin WASM 热加载

Plugin WASM 文件也支持热加载——无需重启 ShopClaw：

```rust
// crates/shopclaw-core/src/plugin/registry.rs

pub struct PluginRegistry {
    plugin_dir: PathBuf,       // ~/.shopclaw/plugins/
    plugins: RwLock<HashMap<String, Arc<PluginSandbox>>>,
    watcher: notify::RecommendedWatcher,
}

impl PluginRegistry {
    /// 监听 plugin_dir，有新 .wasm 文件或文件变化时自动加载/重载
    pub fn watch(&mut self) -> Result<()> {
        use notify::{Watcher, RecursiveMode};

        let plugins = self.plugins.clone();
        let browser = self.browser.clone();

        self.watcher.watch(&self.plugin_dir, RecursiveMode::NonRecursive)?;

        // 文件变化回调
        // - 新增 .wasm → 加载并注册
        // - 修改 .wasm → 卸载旧的，加载新的
        // - 删除 .wasm → 卸载
        Ok(())
    }
}
```

用户可以直接把新的 `.wasm` 文件丢进 `~/.shopclaw/plugins/`，ShopClaw 自动检测并加载，无需重启。

### 16.10 配置总览

```toml
# ~/.shopclaw/config.toml

[selector_sync]
# Selector 远程仓库 URL（默认 GitHub raw）
registry_url = "https://raw.githubusercontent.com/tokligence/ShopClaw/main/registry"
# 同步间隔（启动时立即同步一次，之后按间隔）
sync_interval = "6h"
# 是否启用后台同步（禁用则仅用本地 + LLM）
enabled = true
# 回报 LLM 发现的新 selector 给社区
contribute_discoveries = false

[llm]
# LLM 用于 selector 发现的兜底
provider = "anthropic"
model = "claude-sonnet-4-20250514"
api_key_env = "ANTHROPIC_API_KEY"

[plugins]
# Plugin 目录，支持热加载
plugin_dir = "~/.shopclaw/plugins"
# 是否自动检查 Plugin 更新（从 GitHub releases）
auto_update_plugins = true
```
