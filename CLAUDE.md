# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
cargo build                                        # Build
cargo test                                         # Run all tests
cargo test --all-features                          # Run tests with tracing feature
cargo test test_name                               # Run a single test by name
cargo clippy --all-features -- -D warnings         # Lint
cargo fmt --check                                  # Check formatting
cargo fmt --all                                    # Auto-format
cargo doc --no-deps --all-features                 # Build docs (CI uses RUSTDOCFLAGS="-D warnings")
cargo bench                                        # Run benchmarks (slow - avoid unless needed)
cargo deny check                                   # License/advisory audit
```

MSRV: **1.88.0** (Rust edition 2024, enables let-chains)

## Architecture

Route-based rate limiting middleware for `reqwest` via `reqwest-middleware`. Requests flow through `RateLimitMiddleware::handle()` → pre-extract URL components → iterate all routes → check each matching route's limits via GCRA → delay/error/pass.

### Key modules

- **`middleware.rs`** — `RateLimitMiddleware` implements `reqwest_middleware::Middleware`. Owns shared state as `Arc<[GcraState]>` (flat pre-allocated array indexed by route offsets). Hot-path optimized: URL components extracted once, `now_nanos()` stays in u64, rate-limited branch is `#[cold]`.
- **`gcra.rs`** — Lock-free GCRA algorithm using `AtomicU64` compare-exchange loop. Returns `Ok(())` or `Err(wait_duration)`.
- **`types.rs`** — `Route` (host/method/path matching), `RateLimit` (precomputed emission interval + window in nanos), `ThrottleBehavior` (Delay or Error).
- **`builder.rs`** — Fluent builder API. `RateLimitBuilder` → `HostBuilder`/`RouteBuilder`. `build()` precomputes a flat offset table mapping route index → position in the states array.
- **`error.rs`** — `RateLimitError::RateLimited(Duration)` with `reqwest_middleware::Error` integration.

### Critical design decisions

- **All matching routes apply** — routes are checked in order and every match has its limits enforced (enables layering general + specific limits).
- **Path matching uses segment boundaries** — `/order` matches `/order/123` but NOT `/orders`.
- **State is a flat `Arc<[GcraState]>` array** — key space is bounded at build time, so no map/lock needed. `route_offsets[i]` gives the start index for route `i`'s states.
- **Jitter on delay** — 0-50% random jitter added to sleep duration to prevent thundering herd.

### Feature flags

- `tracing` (optional) — emits `tracing::warn!` when catch-all routes precede specific routes in the builder.

### Test structure

- Unit tests: in-module (`src/lib.rs`, `src/types.rs`, `src/gcra.rs`, `src/builder.rs`)
- Integration tests: `tests/integration.rs` using `wiremock` for HTTP mocking
- Benchmarks: `benches/throughput.rs` using criterion (route matching, stacked limits, concurrent throughput)
