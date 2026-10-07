# ROADMAP.md — work top to bottom, one milestone at a time

Tick boxes as you go. Each milestone ends green on the full check suite (see AGENTS.md).

## M0 — Skeleton
- [ ] Cargo workspace with all crates (empty), `rust-toolchain.toml`, `rustfmt.toml`, `clippy.toml`, `deny.toml`
- [ ] GitHub Actions CI (fmt, clippy, test, deny)
- [ ] `orbit-theme`: seed → M3 dark tokens; unit tests
- [ ] Slint window with Material dark style, `Theme` global, three-pane layout placeholders
- [ ] `--mock` flag wiring; `docs/DECISIONS.md` created with pinned versions
**Done when:** window opens with correct tonal surfaces and CI is green.

## M1 — Server directory → tree
- [ ] `orbit-ovpn` parser + dangerous-directive report
- [ ] Resolver (dir layout, filename patterns, ISO/IATA tables, metadata comments)
- [ ] Scanner with `rayon`, `notify` live rescan, problems list
- [ ] Sidebar: virtualised Country → City → Server tree, search, flags
- [ ] Synthetic 2,000-file test-data generator + perf benchmark
**Done when:** F1 acceptance passes; UI stays responsive during scan.

## M2 — Profiles & storage
- [ ] `orbit-store`: settings/profiles TOML with atomic writes + versioning
- [ ] Keyring wrapper; profile editor dialog; favourites; recents
- [ ] First-run wizard (directory, consent, seed colour)
**Done when:** F2 acceptance passes.

## M3 — Mock connect + state machine
- [ ] `orbit-core` state machine + `Command/Event/AppState`
- [ ] `MockBackend` simulating states, bytecount, failures
- [ ] Hero card, status chips, connect/disconnect/cancel, snackbars
- [ ] Stats engine (EMA speeds, totals, 60 s ring buffer) + bottom strip + sparkline
- [ ] Session timer
**Done when:** the full UI works end-to-end in `--mock`; F4/F5 acceptance on mock.

## M4 — Real OpenVPN
- [ ] `orbit-mgmt` client + fake-server integration tests
- [ ] `orbit-helper` + polkit policy + validation (SECURITY.md checklist) + fuzz targets
- [ ] `RealBackend`: spawn, credentials via management, state mapping, graceful stop/kill
- [ ] 2FA/challenge dialog, auto-reconnect with backoff, re-attach to running session
- [ ] DNS handling (systemd-resolved first)
**Done when:** one-click connect to a real test server works from a cold start; F3 acceptance.

## M5 — Geo + 3D globe
- [ ] `orbit-geo` provider chain, cache, consent, offline MaxMind-format provider
- [ ] Origin lookup before connect; exit lookup via tunnel; leak/identical-IP warning
- [ ] `orbit-globe`: sphere, texture, atmosphere, markers, camera, slerp arc, easing, interaction
- [ ] wgpu → Slint texture integration; 2D fallback renderer
- [ ] Overlay chips, reduced-motion support
**Done when:** F6 acceptance; matches reference screenshot layout (F7).

## M6 — Polish & extras
- [ ] Tray (ksni), notifications, launch-on-login, start minimised
- [ ] Settings side sheet complete; log viewer; diagnostics copy
- [ ] Kill switch (nftables via helper) + "restore networking" escape hatch
- [ ] Accessibility pass; keyboard shortcuts; i18n scaffolding
- [ ] Packaging: AppImage, PKGBUILD, polkit policy install, `.desktop`

## M7 — Hardening & release
- [ ] Security review against SECURITY.md checklist; fuzz 1h+ clean
- [ ] Performance budgets met; idle CPU/RAM measured
- [ ] Docs: README, install guide, troubleshooting; v0.1.0 tag

## Backlog (P2)
Server latency probing & real "Fastest", split tunnelling, WireGuard backend, Windows/macOS ports, provider-specific metadata import (server load), profile import/export, multi-hop.
