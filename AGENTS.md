# AGENTS.md — Orbit VPN

Guidance for any AI coding agent working in this repo. Humans: this doubles as contributor docs.

## Project snapshot
Rust workspace. Material You dark-theme GUI (Slint) for OpenVPN. Multi-profile, one-click connect, country/city server tree built from a single directory of `.ovpn` files, live up/down stats, session timer, 3D globe (wgpu) showing real vs VPN location.

## Repo map
```
orbit/
├─ AGENTS.md                 # this file
├─ docs/                     # FEATURES, TECH_STACK, ARCHITECTURE, UI_DESIGN, SECURITY, TESTING, ROADMAP, DECISIONS
├─ crates/
│  ├─ orbit-core/            # domain types, events, state machine (no IO)
│  ├─ orbit-ovpn/            # .ovpn parser, dir scanner, country/city resolver
│  ├─ orbit-mgmt/            # OpenVPN management-interface client (async)
│  ├─ orbit-session/         # spawns helper, owns connection lifecycle, stats, timer
│  ├─ orbit-geo/             # IP geolocation providers + cache + offline fallback
│  ├─ orbit-store/           # settings/profiles (TOML), keyring, history
│  ├─ orbit-globe/           # wgpu globe renderer (+ 2D fallback)
│  ├─ orbit-theme/           # Material 3 tokens from seed colour
│  ├─ orbit-ui/              # Slint UI + view-models glue
│  ├─ orbit-helper/          # privileged helper binary (tiny, audited)
│  └─ orbit-app/             # main binary, wiring only
├─ assets/                   # globe textures, flag SVGs, IATA table, icons
└─ packaging/                # polkit policy, .desktop, AppImage/PKGBUILD
```

## Commands
```
cargo run -p orbit-app                       # run the app
cargo fmt --all
cargo clippy --workspace --all-targets -- -D warnings
cargo test --workspace
cargo deny check                             # licences + advisories
cargo nextest run                            # optional faster tests
```
Run all four of fmt / clippy / test / deny before declaring any task done.

## Rules
### Architecture
- Dependency direction: `app → ui → session/store/geo/globe/theme → ovpn/mgmt → core`. `orbit-core` has zero IO and zero async runtime dependencies.
- UI talks to the backend only through `Command` (UI→backend) and `Event` / `AppState` snapshot (backend→UI) channels defined in `orbit-core`. No shared mutable state, no `Arc<Mutex<…>>` in UI code.
- The connection lifecycle is an explicit state machine (`Idle → Preparing → Connecting → Authenticating → GettingConfig → AssigningIp → Connected → Disconnecting → Idle`, plus `Failed(reason)`, `Reconnecting`). Illegal transitions are unrepresentable or return `Err`.

### Code style
- `thiserror` for library crates, `anyhow` only in `orbit-app`/binaries.
- No `unwrap()`/`expect()`/`panic!` in non-test code; use `?` and typed errors. `expect` allowed for provable invariants with a comment.
- `#![forbid(unsafe_code)]` in every crate except where wgpu/FFI forces otherwise (document why).
- Public items documented. Modules < ~400 lines; split otherwise.
- Use `tracing` (never `println!`). Never log credentials, tokens, full `.ovpn` contents, or the user's real IP at `info` or above.
- Async: tokio. Never `block_on` in the UI thread. Every spawned task has an owner and a cancellation path.

### Security (see docs/SECURITY.md)
- GUI never runs as root. Privileged ops only via `orbit-helper` (polkit `pkexec`/D-Bus).
- Helper validates: config path is inside the configured `.ovpn` dir (canonicalised, no symlink escape), directive allow/deny list, management socket path owned by the invoking user.
- Treat every `.ovpn` as hostile. Deny `up`, `down`, `route-up`, `route-pre-down`, `tls-verify`, `ipchange`, `learn-address`, `auth-user-pass-verify`, `client-connect`, `client-disconnect`, `plugin`, `script-security >1`, `setenv` of dangerous vars unless the user explicitly opts in per profile.
- Secrets: `keyring` crate only. Zeroize in-memory secrets (`secrecy` + `zeroize`).

### UI / UX
- Follow `docs/UI_DESIGN.md`. Use Material 3 tokens only; no hard-coded hex colours in `.slint` files outside the theme module.
- Dark theme is the default and the only theme in v1. Dynamic colour from a seed colour (default: teal-green) via `orbit-theme`.
- Minimum window 960×600; layout must also be usable at 720 wide (collapse the right rail).
- Every interactive element is keyboard reachable and has an accessible label.
- Animations ≤ 300 ms except the globe; honour `prefers-reduced-motion` equivalent (setting).

### Testing
- Pure logic (parser, resolver, state machine, stats math) has unit tests and, where useful, `proptest` property tests.
- Management-interface client is tested against a **fake OpenVPN server** (tokio TCP/Unix listener replaying scripted transcripts) in `orbit-mgmt/tests`.
- A `MockSession` backend lets the UI run without root/openvpn: `cargo run -p orbit-app -- --mock`.
- Parser fuzzing target in `orbit-ovpn/fuzz`.

### Git
- Conventional commits (`feat:`, `fix:`, `refactor:`, `docs:`, `test:`, `chore:`).
- One logical change per commit. Do not commit `.ovpn`, keys, or real server lists; use `testdata/` with synthetic files.

## Workflow for a task
1. Read the relevant doc section + the milestone in `docs/ROADMAP.md`.
2. Write a short plan (files, types, tests). Keep it in the PR/commit description.
3. Write tests first for logic-heavy code; implement; run the check suite.
4. Update docs/ROADMAP checkboxes and `docs/DECISIONS.md` if you deviated.
5. Summarise: what changed, how verified, what is next.

## Boundaries
**Always:** run the check suite, add tests, keep docs in sync, ask before adding a new dependency that is not in `TECH_STACK.md` (add an ADR line instead if you cannot ask).
**Ask first:** changing the UI toolkit, adding network calls beyond the geo-IP providers, changing the helper's privileges, adding telemetry (default answer: no).
**Never:** run the GUI as root, store plain-text credentials, execute scripts from `.ovpn` files by default, add telemetry/analytics, commit secrets or real VPN configs, disable TLS verification for geo lookups.

## Known sharp edges
- Slint version features (wgpu texture import, Material style) move between releases — check the current Slint docs before relying on them and record the pinned version in `docs/DECISIONS.md`.
- OpenVPN 2.6 vs 2.7 differ in DCO and `--dns` handling; detect the version at startup via `openvpn --version` and adapt.
- DNS handling on Linux depends on resolver (systemd-resolved, resolvconf, NetworkManager). See `docs/ARCHITECTURE.md` §DNS.
