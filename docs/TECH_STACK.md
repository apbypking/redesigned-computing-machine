# TECH_STACK.md

Pinned choices with reasons. Changing any row requires an ADR in `docs/DECISIONS.md`. Verify versions against crates.io when scaffolding and pin them in `Cargo.toml` / `Cargo.lock`.

## Decision summary
| Concern | Choice | Why |
|---|---|---|
| Language | Rust (stable, 2021/2024 edition) | Requirement |
| GUI toolkit | **Slint** (`slint`, `slint-build`) | Ships a Material 3 widget style with a dark scheme, declarative `.slint` UI, GPU or software rendering, good Wayland support, small footprint |
| Material You theming | `material-colors` (Material Color Utilities port) + custom `Theme` global in Slint | Generates the M3 tonal palette from a seed colour; Slint `Palette`/globals consume it |
| 3D globe | **wgpu** + `glam` + `bytemuck`, rendered to a texture and imported into Slint | Real 3D sphere, arc, glow; Slint can display wgpu textures (feature flag, verify current version) |
| Globe fallback | Orthographic 2D projection drawn with Slint `Path`/`Image` | Used when no GPU adapter or wgpu import unavailable |
| Async runtime | `tokio` (multi-thread) | Process, socket, timers, HTTP |
| OpenVPN control | System `openvpn` binary + **management interface** over Unix socket | Authoritative state, bytecount, clean signalling |
| Privilege escalation | `orbit-helper` binary via **polkit** (`pkexec` first; D-Bus activation later) | GUI stays unprivileged |
| HTTP | `reqwest` (rustls, no default OpenSSL) | Geo-IP lookups |
| Geo-IP online | Provider trait; default `ipwho.is` / `ipapi.co`, HTTPS only, fallback chain | No API key required for basic use |
| Geo-IP offline | DB-IP Lite or GeoLite2 City via `maxminddb` (user-supplied DB path) | Works without network calls, resolves server `remote` IPs |
| Serialization | `serde`, `toml` (settings/profiles), `serde_json` (cache/history) | Human-editable config |
| Paths | `directories` | XDG dirs |
| FS watching | `notify` (+ debounce) | Live rescan of the `.ovpn` directory |
| Secrets | `keyring` (Secret Service / KWallet), `secrecy`, `zeroize` | No plain-text creds |
| Tray | `ksni` (StatusNotifierItem) on Linux, `tray-icon` elsewhere | Hyprland/Waybar tray support |
| D-Bus | `zbus` | Tray, notifications, optional polkit/NM integration |
| Notifications | `notify-rust` | Connect/disconnect/failure toasts |
| Errors | `thiserror` (libs), `anyhow` (bins) | Standard |
| Logging | `tracing`, `tracing-subscriber`, `tracing-appender` | Structured, redactable |
| Images/flags | `resvg`/`usvg` or pre-baked PNG atlas; Natural Earth textures (public domain) | Flags + globe texture |
| IATA/city table | Bundled CSV (`assets/iata.csv`) parsed at build time | Resolve filenames like `us-nyc-12` |
| Testing | `cargo test`, `cargo nextest`, `proptest`, `insta` (snapshots), `cargo-fuzz` | See TESTING.md |
| Supply chain | `cargo-deny`, `cargo-audit` | Licences, advisories |
| CI | GitHub Actions: fmt, clippy, test, deny, build matrix | |
| Packaging | AppImage + Arch `PKGBUILD` first; `.deb` later | Target includes Arch/Omarchy-style systems |

## Why Slint (and the fallbacks)
- Material 3 look out of the box → fastest route to "Material You". If the built-in style lacks a needed component (navigation rail, segmented button, FAB), build it in `ui/components/` using tokens from `Theme`.
- **Licensing:** Slint is GPLv3 / royalty-free / commercial. Confirm the royalty-free or GPL terms fit before shipping and record in `docs/DECISIONS.md`.
- **Fallback A:** Iced + custom M3 theme + wgpu shader widget (globe integrates natively, more custom widget work).
- **Fallback B:** egui + `egui_wgpu` (fast to prototype, weakest Material fidelity).
- Do **not** use Tauri/webviews: requirement is a Rust-native GUI.

## OpenVPN integration details
- Spawn (via helper): `openvpn --config <file> --management <unix-socket> unix --management-hold --management-query-passwords --auth-nocache --script-security 1` plus any allow-listed extras.
- Management commands used: `state on`, `bytecount 1`, `log on`, `hold release`, `username "Auth" …`, `password "Auth" …`, `signal SIGTERM`.
- Notifications parsed: `>STATE:`, `>BYTECOUNT:`, `>LOG:`, `>PASSWORD:`, `>FATAL:`, `>HOLD:`.
- Minimum supported OpenVPN: 2.5; target 2.6+. Detect DCO (data channel offload): bytecount works but verify.

## Data layout (XDG)
```
~/.config/orbit/settings.toml     # ovpn_dir, theme seed, geo provider, behaviour flags
~/.config/orbit/profiles.toml     # saved connections (see ARCHITECTURE.md)
~/.local/share/orbit/history.json # sessions: start, duration, bytes, server
~/.cache/orbit/geo.json           # cached lookups (TTL), last real location
$XDG_RUNTIME_DIR/orbit/mgmt.sock  # management socket (0600)
Secret Service: service "orbit-vpn", account "<profile-id>"
```

## Dev environment
- Rust stable + `clippy`, `rustfmt`; `openvpn` installed; `polkit`; Vulkan/GL drivers for wgpu; `fontconfig`.
- Optional: `cargo-nextest`, `cargo-deny`, `cargo-fuzz` (nightly), `cargo-watch`.
