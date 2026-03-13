# ShopClaw Design Review

## Review Summary

| Dimension | Verdict | Key Issue |
|-----------|---------|-----------|
| **Coverage** | Good, with gaps | Login-required sites, mobile-only sites, multi-step checkout not standardized |
| **Generality** | Architecture general, types too specific | WIT types are shopping-only; booking/delivery would need new types |
| **Security** | Solid foundation, 4 gaps | No plugin signing, token in plaintext, no WASM resource limits, audit tamperable |
| **Future-proofing** | Good near-term, fragile long-term | Mobile browsers unsupported, single-browser (Chrome only), MCP stability unknown |

---

## 1. Coverage Gaps

### 1.1 Login-Required Sites

**Problem**: Many sites require login to see prices or add to cart. The design assumes the user is already logged in via their Chrome profile, but doesn't handle:
- Sites that force re-login after inactivity
- Sites that detect automation and force re-auth
- Multi-factor authentication popups during checkout

**Recommendation**: Add a `login_check` step to the Plugin WIT interface:

```wit
/// Check if user is logged into the site. Returns login status.
check-login: func(tab-id: u64) -> login-status;

enum login-status {
    logged-in,
    login-required,
    mfa-required,
}
```

When `login-required`, Plugin pauses and returns `needs_user_action: true` with prompt "Please log in to {site} in the browser tab that just opened."

### 1.2 Mobile-Only Sites

**Problem**: Some merchants are mobile-first (Instagram Shop, Pinduoduo, Shopee). Their desktop sites are limited or redirect to mobile apps.

**Current workaround**: Plugin can set mobile User-Agent and viewport via CDP:
```rust
browser::evaluate(tab_id, "navigator.userAgent = 'Mozilla/5.0 (iPhone...)'");
```

But this is fragile and may trigger additional risk controls.

**Recommendation**: Not a v1 priority. Document as known limitation. For v2, consider mobile browser support (see Future-proofing section).

### 1.3 Multi-Step Checkout

**Problem**: Checkout flows vary wildly between sites:
- Amazon: 1-click or cart → shipping → payment → confirm
- Taobao: cart → confirm order → pay via Alipay (redirects to separate domain)
- JD: cart → confirm → pay (in-page)

Current design has no abstraction for multi-step workflows.

**Recommendation**: Define a `checkout-flow` WIT type:

```wit
enum checkout-step {
    view-cart,
    select-shipping,
    select-payment,
    review-order,
    confirm-payment,  // BLOCKED — must be done by user
}

record checkout-state {
    current-step: checkout-step,
    total: f64,
    currency: string,
    items: list<cart-item>,
}
```

Plugin navigates through steps until hitting `confirm-payment`, then hands control to user. This standardizes the flow while allowing per-site implementation.

### 1.4 Coupon / Promo Code Entry

**Problem**: Not mentioned in the design. Users expect shopping assistants to find and apply coupons (this is what Honey does).

**Recommendation**: Add coupon-related tools to plugin manifest:

```json
{
  "name": "amazon_find_coupons",
  "description": "Search for available coupons for the current cart",
  "risk_level": "read"
},
{
  "name": "amazon_apply_coupon",
  "description": "Apply a coupon code to the cart",
  "risk_level": "write"
}
```

Coupon sources: site's own coupon page, browser extension coupon databases (public APIs), user-provided codes.

### 1.5 Multi-Currency

**Problem**: Cross-site comparison (Amazon US vs JD China) requires currency conversion. The `Price` type has a `currency` field but no conversion logic.

**Recommendation**: Add a host-provided currency conversion function:

```wit
/// Convert amount from one currency to another using cached exchange rates.
convert-currency: func(amount: f64, from: string, to: string) -> result<f64, string>;
```

Host fetches exchange rates periodically from a free API (e.g., exchangerate-api.com) and caches locally.

---

## 2. Generality Issues

### 2.1 WIT Types Are Shopping-Specific

**Problem**: The WIT interface defines `Product`, `SearchResult`, `CartSummary` — all shopping concepts. If someone wants to build a plugin for:
- Flight booking (Skyscanner, Google Flights)
- Hotel booking (Booking.com)
- Food delivery (DoorDash, Meituan)
- Bill payment (utilities, subscriptions)

...they'd need completely different return types.

**Current architecture is general** (MCP + WASM + Browser Bridge), but the **type system is narrow**.

**Recommendation**: Split WIT into layers:

```
shopclaw-core.wit          # Generic: browser ops, selectors, risk levels
  └── shopclaw-shopping.wit  # Shopping-specific: Product, Cart, Price
  └── shopclaw-booking.wit   # (future) Booking-specific: Flight, Hotel
  └── shopclaw-delivery.wit  # (future) Delivery-specific: Restaurant, Order
```

Plugins declare which WIT world they implement. Core only depends on `shopclaw-core.wit`.

### 2.2 Plugin Discovery Beyond Shopping

**Problem**: Plugin registry is organized by shopping site (`registry/amazon/`, `registry/jd/`). Non-shopping plugins don't fit this structure.

**Recommendation**: Add category to manifest:

```json
{
  "name": "shopclaw-plugin-skyscanner",
  "category": "booking",
  "site": "skyscanner.com"
}
```

Registry structure:
```
registry/
├── shopping/
│   ├── amazon/
│   └── jd/
├── booking/
│   └── skyscanner/
└── delivery/
    └── doordash/
```

---

## 3. Security Issues

### 3.1 [HIGH] No Plugin Code Signing

**Problem**: Plugins are `.wasm` files loaded from `~/.shopclaw/plugins/`. Any process that can write to this directory can inject a malicious plugin that gets full browser control through the Browser Bridge API.

**Attack vector**:
```
Malicious app writes evil.wasm to ~/.shopclaw/plugins/
  → ShopClaw file watcher auto-loads it
  → evil.wasm calls browser::open_tab("bank.com")
  → evil.wasm reads DOM containing account balance
  → evil.wasm returns data to Agent (or worse, performs transfers)
```

**Mitigation**:

```rust
// Plugin loading with signature verification
pub fn load_plugin(wasm_path: &Path) -> Result<PluginSandbox> {
    let wasm_bytes = std::fs::read(wasm_path)?;
    let sig_path = wasm_path.with_extension("wasm.sig");
    let sig_bytes = std::fs::read(&sig_path)?;

    // Verify ed25519 signature against embedded public key
    let public_key = ed25519_dalek::VerifyingKey::from_bytes(&SHOPCLAW_PUBLIC_KEY)?;
    let signature = ed25519_dalek::Signature::from_bytes(&sig_bytes)?;
    public_key.verify_strict(&wasm_bytes, &signature)?;

    // Only load if signature valid
    PluginSandbox::load_verified(&wasm_bytes)
}
```

- Official plugins signed with ShopClaw project key
- Community plugins: user must explicitly trust (`shopclaw plugins trust <plugin>`)
- Unsigned plugins rejected by default

### 3.2 [HIGH] No WASM Resource Limits

**Problem**: A malicious or buggy plugin can:
- Infinite loop → blocks ShopClaw forever
- Allocate unbounded memory → OOM kills ShopClaw
- Open hundreds of tabs → browser becomes unresponsive

**Mitigation**:

```rust
// Set execution fuel limit (CPU budget)
let mut config = Config::new();
config.consume_fuel(true);

let mut store = Store::new(&engine, state);
store.set_fuel(1_000_000)?;  // ~1 million instructions per invocation

// Set memory limit
let memory_type = MemoryType::new(1, Some(256));  // 1-256 pages (16MB max)
```

Add per-invocation timeout:
```rust
tokio::time::timeout(Duration::from_secs(30), sandbox.invoke(tool, params)).await
```

### 3.3 [MEDIUM] Token Stored in Plaintext

**Problem**: `~/.shopclaw/relay-token` contains the WebSocket auth token as plaintext. Any local process can read it (default file permissions are 644).

**Current mitigation**: File permissions set to `0600` (user-only read). This is OK on single-user systems but insufficient on shared machines.

**Better approach**:

| OS | Storage | API |
|----|---------|-----|
| macOS | Keychain | `security add-generic-password` |
| Linux | Secret Service (GNOME Keyring) | `libsecret` / `secret-service` crate |
| Windows | Credential Manager | `windows-credentials` crate |

Fallback to file-based token on systems without keyring, with warning to user.

### 3.4 [MEDIUM] Audit Log Tamperable

**Problem**: `~/.shopclaw/audit.log` is a regular file. User or malicious process can delete/modify it.

**Mitigation options**:
1. **Hash chain**: Each log entry includes the hash of the previous entry. Tampering breaks the chain.
   ```json
   {"seq": 42, "prev_hash": "a1b2c3...", "timestamp": "...", "tool": "...", "hash": "d4e5f6..."}
   ```
2. **OS log integration**: Write to syslog/journald (harder to tamper without root).
3. **Append-only flag**: `chattr +a audit.log` on Linux (requires root to set).

Recommendation: Hash chain for v1 (simplest), OS log integration for v2.

### 3.5 [MEDIUM] WebSocket Should Use Unix Domain Socket

**Problem**: localhost TCP WebSocket is accessible to any local process. Even with token auth, it's an unnecessary attack surface.

**Recommendation**: Use Unix domain socket instead of TCP:
```
~/.shopclaw/relay.sock  (permissions 0600)
```

Benefits:
- File permission enforcement by OS kernel
- No port conflicts
- Not visible via `netstat` (less obvious attack target)
- Slightly faster (no TCP overhead)

Fallback to TCP localhost for Windows (no Unix sockets).

### 3.6 [LOW] Extension ID Validation

**Problem**: Any Chrome extension on the same machine can attempt to connect to ShopClaw's WebSocket (if it discovers the port/token).

**Mitigation**: ShopClaw should validate the Chrome extension ID during handshake:

```rust
// During WebSocket handshake, extension sends its ID
let expected_extension_id = "abcdefghijklmnopqrstuvwxyz123456";  // from Chrome Web Store
if msg.extension_id != expected_extension_id {
    return Err(BrowserError::UnauthorizedExtension);
}
```

---

## 4. Future-Proofing Issues

### 4.1 [HIGH] Desktop-Only, No Mobile Support

**Problem**: Chrome extensions don't exist on mobile browsers (Chrome for Android, Safari for iOS). The current architecture cannot support mobile shopping.

**Impact**: Significant portion of shopping happens on mobile. Users expect to use ShopClaw from their phone.

**Options for future mobile support**:

| Approach | Effort | Coverage |
|----------|--------|----------|
| Companion mobile app with accessibility service | High | Android only |
| React Native wrapper with WebView | Medium | Both, but limited |
| Cloud-based browser (headless Chrome on server) | Medium | Both, but breaks "local-only" principle |

**Recommendation**: Acknowledge as v2+ scope. For v1, clearly document "desktop only" in README.

### 4.2 [MEDIUM] Chrome-Only, No Firefox/Safari

**Problem**: Chrome extension API (`chrome.debugger`) is Chrome-specific. Firefox uses different extension APIs, Safari has its own format entirely.

**Options**:

| Approach | Pros | Cons |
|----------|------|------|
| Maintain separate extensions per browser | Best UX | 3x maintenance |
| Use WebDriver/Playwright directly | Browser-agnostic | Requires debug port, conflicts with user |
| Use Chrome-only, recommend Chrome | Simple | Excludes Firefox/Safari users |

**Recommendation**: Chrome-only for v1. If demand exists, add Firefox support via separate extension in v2. Safari is low priority (small market share for power users).

### 4.3 [MEDIUM] MCP Protocol Stability

**Problem**: MCP is still evolving. If the protocol makes breaking changes, ShopClaw's MCP server needs updating.

**Mitigation**:
- Implement MCP version negotiation in `initialize` response
- Abstract MCP protocol behind an internal trait so the transport can be swapped
- Pin to a specific MCP protocol version, add others as needed

### 4.4 [LOW] Selector Registry Single Point of Failure

**Problem**: If GitHub is down or blocks raw.githubusercontent.com (happens in some countries), selector sync fails.

**Mitigation options**:
1. **Multiple mirrors**: Add fallback URLs (GitLab mirror, Cloudflare Pages)
2. **Bundled defaults**: Ship a snapshot of selectors in the binary, use as Layer 0
3. **P2P sync**: Overkill for v1

**Recommendation**: Bundle selector snapshot in binary + add mirror URL config option.

### 4.5 [LOW] Multi-Modal Support

**Problem**: Current design is text-only (JSON in, JSON out). Future Agents may want:
- Image results (product photos)
- Voice interaction
- Video previews

**Recommendation**: MCP supports multi-modal content blocks. Plugin return type should support:
```json
{
  "content": [
    {"type": "text", "text": "AirPods Pro - $249"},
    {"type": "image", "data": "base64...", "mimeType": "image/png"}
  ]
}
```

Add this to the Router's response serialization when needed, but don't block v1 on it.

---

## 5. Action Items

### Must-Have (Before v1)

| # | Item | Section |
|---|------|---------|
| 1 | Add WASM fuel + memory limits | 3.2 |
| 2 | Add plugin ed25519 signing | 3.1 |
| 3 | Switch WebSocket to Unix domain socket | 3.5 |
| 4 | Add `login-check` to Plugin WIT | 1.1 |
| 5 | Add per-invocation timeout (30s) | 3.2 |
| 6 | Bundle selector snapshot in binary | 4.4 |

### Should-Have (v1.1)

| # | Item | Section |
|---|------|---------|
| 7 | Token storage in OS keyring | 3.3 |
| 8 | Audit log hash chain | 3.4 |
| 9 | Currency conversion host function | 1.5 |
| 10 | Checkout flow abstraction | 1.3 |
| 11 | Coupon search tools | 1.4 |
| 12 | Extension ID validation | 3.6 |

### Nice-to-Have (v2+)

| # | Item | Section |
|---|------|---------|
| 13 | WIT layer split (shopping/booking/delivery) | 2.1 |
| 14 | Firefox extension | 4.2 |
| 15 | Mobile support architecture | 4.1 |
| 16 | Multi-modal MCP responses | 4.5 |
| 17 | Selector mirror URLs | 4.4 |
| 18 | MCP version negotiation | 4.3 |
