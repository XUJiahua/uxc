# Caller-Side Environment Variable Resolution for Auth Credentials

> 目标路径: `/Users/administrator/my/uxc/docs/decisions/caller-env-auth-resolution.md`

This note proposes a change to how `env:`-sourced credential secrets are resolved when the daemon
is involved, enabling per-request dynamic auth headers driven by the caller's environment.

## Problem

When a credential uses `--secret-env FINDER_TOKEN`, the `{{secret}}` template is resolved via
`std::env::var("FINDER_TOKEN")`. This call executes inside the **daemon process**, which is a
long-lived singleton. The daemon inherits its environment at startup and never sees changes to the
caller's shell environment.

```
caller shell (FINDER_TOKEN=abc)
    → uxc client process (can see FINDER_TOKEN=abc)
        → Unix socket → RuntimeInvokeRequest
            → daemon process (cannot see caller's FINDER_TOKEN)
                → resolve_secret() → std::env::var("FINDER_TOKEN") → ❌ not set or stale
```

This makes `env:`-sourced credentials unusable for multi-tenant scenarios where different
invocations need different secret values (e.g., per-user session tokens passed via environment
variables).

## Use Case

A SaaS agent platform runs multiple user sessions on a single machine. Each session sets a
per-user token via environment variable before invoking a linked CLI:

```bash
FINDER_TOKEN=user_a_jwt finder-proxy-cli 'get:/eval'
FINDER_TOKEN=user_b_jwt finder-proxy-cli 'get:/eval'
```

The credential is configured once at deploy time:

```bash
uxc auth credential set finder-tenant \
  --auth-type api_key \
  --secret-env FINDER_TOKEN \
  --header "Authorization=Bearer {{secret}}"

uxc auth binding add --id finder-proxy-binding \
  --host localhost:9090 \
  --path-prefix /api/proxy/finder \
  --scheme http \
  --credential finder-tenant
```

Expected behavior: each invocation resolves `$FINDER_TOKEN` from its own environment and injects
the corresponding `Authorization` header. Currently this does not work because the daemon cannot
see the caller's environment.

## Proposed Solution

**Resolve `env:`-sourced secrets on the client side and pass the rendered headers to the daemon
via the existing `request_headers` field in `RuntimeInvokeOptions`.**

### Why `request_headers`

`RuntimeInvokeOptions` already has a `request_headers: HashMap<String, String>` field, and the
OpenAPI adapter already injects these headers into outgoing HTTP requests (`openapi.rs:2146`).
The field is currently always empty (`HashMap::new()`) because no client code populates it.

### Changes

#### 1. Client side (`src/main.rs`)

When building `RuntimeInvokeOptions`, if the matched auth binding's credential has an `env:`
secret source, resolve the credential's auth headers in the client process and populate
`request_headers`:

```rust
// Before sending RuntimeInvokeRequest to daemon
let mut request_headers = HashMap::new();

if let Ok(Some(profile)) = auth::resolve_auth_for_endpoint(&url, cli.auth.clone()) {
    if profile.has_env_sourced_secret() {
        // Resolve in client process where caller's env vars are visible
        if let Ok(headers) = profile.resolved_auth_headers() {
            for (name, value) in headers {
                request_headers.insert(name, value);
            }
        }
    }
}

// Pass to daemon
RuntimeInvokeOptions {
    request_headers,
    // ... other fields unchanged
}
```

This applies to both the direct-invoke path (~line 1910) and any other path that builds
`RuntimeInvokeOptions`.

#### 2. Auth module (`src/auth/mod.rs`)

Add a helper method to check if a profile's secret comes from an environment variable:

```rust
impl Profile {
    /// Returns true if the credential's secret is sourced from an environment variable.
    pub fn has_env_sourced_secret(&self) -> bool {
        matches!(&self.secret_source, Some(SecretSource::Env { .. }))
    }
}
```

No changes needed to `resolved_auth_headers()` itself — it already calls `std::env::var()`
which works correctly when called in the client process.

#### 3. Daemon side (`src/daemon.rs`)

When `request_headers` is non-empty, the daemon should **skip** its own auth binding resolution
for that request, since the client has already handled it:

```rust
// In invoke_inner() or the adapter dispatch path
let auth_profile = if request.options.request_headers.is_empty() {
    // Normal path: daemon resolves auth
    auth::resolve_auth_for_endpoint(&request.endpoint, request.options.auth.clone())?
} else {
    // Client already resolved env-based auth into request_headers
    None
};
```

#### 4. OpenAPI adapter (`src/adapters/openapi.rs`)

**No changes needed.** The adapter already applies `self.request_headers` to every outgoing
request at line 2146:

```rust
for (name, value) in &self.request_headers {
    req = req.header(name, value);
}
```

### Behavior Matrix

| Secret source | Daemon mode | Resolution location | Mechanism |
|---------------|-------------|--------------------|----|
| `literal:` | daemon | daemon | existing (unchanged) |
| `env:` | daemon | **client** | client pre-resolves → `request_headers` |
| `op://` | daemon | daemon | existing (unchanged) |
| `env:` | no daemon | client | existing (already works) |
| any | no daemon | client | existing (already works) |

### Backward Compatibility

- If `request_headers` is empty (the current default), behavior is unchanged.
- Credentials with `literal:` or `op://` secrets continue to resolve in the daemon.
- The `--inject-env` mechanism for stdio endpoints is unaffected (different code path).

### Security Considerations

- Secret values transit over the Unix domain socket between client and daemon. This is the same
  trust boundary as `--inject-env` for stdio endpoints, which already passes resolved secrets
  through the socket.
- The daemon socket is user-owned (`~/.uxc/daemon/uxc.sock`) with restricted permissions.
- No secrets are logged or cached in the `request_headers` path.

## Alternatives Considered

### A. Pass caller env snapshot to daemon

Add a `caller_env: HashMap<String, String>` field to `RuntimeInvokeOptions`. The daemon uses
this map instead of `std::env::var()` when resolving `env:` secrets.

Pros: More general (supports future env-dependent features).
Cons: Larger change surface; daemon auth code needs an env override parameter threaded through.

### B. Disable daemon for env-sourced credentials

If the matched credential uses `env:`, bypass the daemon and execute directly in the client.

Pros: Zero daemon changes.
Cons: Loses daemon benefits (connection pooling, schema caching, session reuse).

### C. Restart daemon per invocation

Cons: Defeats the purpose of a daemon. Unacceptable latency.

## Summary of Changes

| File | Change | Lines |
|------|--------|-------|
| `src/main.rs` | Pre-resolve env auth → `request_headers` | ~15 |
| `src/auth/mod.rs` | Add `has_env_sourced_secret()` | ~5 |
| `src/daemon.rs` | Skip auth resolution when `request_headers` populated | ~5 |
| `src/adapters/openapi.rs` | None (already supports `request_headers`) | 0 |

Total: ~25 lines of production code.
