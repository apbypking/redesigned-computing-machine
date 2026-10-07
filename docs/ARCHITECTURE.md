# ARCHITECTURE.md

## Process model
```
┌────────────────────────── orbit-app (user, unprivileged) ──────────────────────────┐
│  Slint UI thread  ◄── AppState snapshots ──  Backend (tokio)                        │
│        │ Commands ───────────────────────►   ├─ Scanner  (orbit-ovpn, notify)        │
│        │                                     ├─ Store    (profiles, settings, keyring)│
│        │                                     ├─ Geo      (orbit-geo)                 │
│   Globe renderer (wgpu texture) ◄─ markers ──├─ Session  (orbit-session)             │
│                                              └─ Mgmt     (orbit-mgmt) ──┐            │
└──────────────────────────────────────────────────────────────────────────┼───────────┘
            pkexec / polkit (validated args)                               │ Unix socket
┌──────────── orbit-helper (root, tiny) ────────────┐                      │ 0600, user-owned
│ validate → spawn `openvpn` → relay exit status    │◄─────────────────────┘
└───────────────────────────────────────────────────┘
```

## Data flow (connect click)
1. UI sends `Command::Connect { profile_id }`.
2. Session resolves profile → server → `.ovpn` path; fetches credentials from keyring.
3. Geo: `Geo::origin()` returns cached/fresh real location (before tunnel). Event `OriginLocated`.
4. Session asks helper to start openvpn with a fresh management socket path in `$XDG_RUNTIME_DIR/orbit/`.
5. Mgmt client connects, sends `state on`, `bytecount 1`, `log on`, `hold release`; answers `>PASSWORD:` prompts from keyring (or emits `Event::NeedCredentials` / `NeedChallenge`).
6. `>STATE:` notifications drive the state machine; on `CONNECTED` capture `connected_at = Instant::now()` and the tunnel IP / remote IP from the state line.
7. Geo: `Geo::exit()` fetches location **through the tunnel**; event `ExitLocated`; globe animates arc.
8. `>BYTECOUNT:` every second → `StatsEngine` → speeds, totals, sparkline ring buffer → included in `AppState`.
9. Disconnect: `signal SIGTERM` over management; helper waits for exit; escalate to SIGKILL after 5 s; cleanup DNS/routes; persist session to history.

## Core types (orbit-core)
```rust
pub struct Server { id: ServerId, path: PathBuf, name: String, country: CountryCode,
                    city: Option<String>, remote: RemoteEndpoint, proto: Proto,
                    needs_auth: bool, tags: Vec<String> }
pub struct Profile { id: ProfileId, name: String, target: Target, creds: CredRef,
                     options: ProfileOptions, favourite: bool }
pub enum Target { Server(ServerId), City(CountryCode, String), Country(CountryCode), Fastest }
pub enum ConnState { Idle, Preparing, Connecting, Authenticating, GettingConfig,
                     AssigningIp, Connected{ since: Instant, tunnel_ip: IpAddr, remote_ip: IpAddr },
                     Reconnecting{ attempt: u32 }, Disconnecting, Failed(FailReason) }
pub struct Stats { down_bps: f64, up_bps: f64, down_total: u64, up_total: u64,
                   history: RingBuf<Sample, 60> }
pub struct GeoPoint { lat: f64, lon: f64, country: CountryCode, city: Option<String>,
                      ip: IpAddr, accuracy: Accuracy }
pub enum Command { Connect(ProfileId), ConnectServer(ServerId), QuickConnect, Disconnect,
                   SetOvpnDir(PathBuf), SaveProfile(Profile), DeleteProfile(ProfileId),
                   ProvideCredentials{..}, Rescan, UpdateSettings(Settings), .. }
pub struct AppState { servers: ServerTree, profiles: Vec<Profile>, conn: ConnState,
                      stats: Stats, origin: Option<GeoPoint>, exit: Option<GeoPoint>,
                      settings: Settings, problems: Vec<ScanProblem> }
```

## Crates in detail
### orbit-ovpn
- `scan(dir) -> ScanResult { servers, problems }` using `walkdir`, parallel parse with `rayon`.
- Parser extracts only what the UI needs: `remote`, `proto`, `auth-user-pass` presence, `dev`, comments metadata, dangerous-directive report. It does **not** need to be a full OpenVPN parser.
- `Resolver` implements the 5-step country/city strategy; tables: ISO countries (names + aliases), IATA → city/country, normalising (case, diacritics, `_`/`-`/`.`).
- Property tests: random filenames never panic; parse(serialise(x)) stable.

### orbit-mgmt
- Line-oriented async codec (`tokio_util::codec::LinesCodec`) with multi-line `>` notification and `SUCCESS:/ERROR:` command-reply correlation (single in-flight command queue).
- Strongly typed `Notification` enum; unknown lines retained as `Raw`.
- Reconnect-to-running-session support: `attach(socket)` sends `state`, `bytecount 1`.

### orbit-session
- Owns the state machine, backoff timer, helper spawning, credential prompts, stats engine, session timer, history writer.
- `trait Backend` implemented by `RealBackend` and `MockBackend` (`--mock`) so the UI is developable without root.

### orbit-geo
- `trait GeoProvider { async fn lookup(&self, ip: Option<IpAddr>) -> Result<GeoPoint> }`; chain with timeouts (3 s each), retries, jitter, TTL cache (origin 24 h or until default-route/IP change; exit per session).
- Public-IP discovery then lookup; offline `maxminddb` provider for server IPs.
- Verify exit ≠ origin; if equal after CONNECTED, surface a **"possible leak / not routed"** warning.
- Privacy: first-run consent for online lookups; setting to disable; origin location blur option.

### orbit-globe
- `Globe::new(device, queue)`, `render(&mut self, t, camera, markers, arc) -> wgpu::Texture`.
- Pipeline: UV-sphere mesh (≥ 64×64), equirectangular day/dark-style texture with city-light overlay, atmosphere rim glow (fresnel in shader), instanced marker quads (billboards), arc as camera-facing triangle strip along a slerp curve with height `h = sin(πt)·k`, animated dash/trail via uniform `progress`.
- Camera: orbit camera (yaw/pitch/distance) with critically-damped easing; `frame_two_points(a, b)` picks yaw/pitch/distance so both points + arc are visible.
- Lat/lon ↔ 3D: `x = cos(lat)·sin(lon)`, `y = sin(lat)`, `z = cos(lat)·cos(lon)`; keep the helper in `orbit-core::geo_math` with unit tests.
- Fallback `Map2d` renders orthographic/equirectangular with same API.

### orbit-theme
- `Theme::from_seed(Argb) -> Tokens` using `material-colors` (dark scheme). Exposes: `primary, on_primary, primary_container, secondary, tertiary, surface, surface_container{,_low,_high,_highest}, on_surface, on_surface_variant, outline, outline_variant, error, success (custom), …`.
- Pushed into a Slint `global Theme` at startup and on change; no colours elsewhere.

### orbit-store
- `settings.toml`, `profiles.toml` atomic writes (tmp + rename), schema `version` field with migrations.
- Keyring wrapper `Secrets::{get,set,delete}(profile_id)` returns `SecretString`.

### orbit-helper
- Single-purpose: `orbit-helper run --config <path> --mgmt <sock> --uid <uid> -- [allow-listed args]`.
- Re-validates everything it receives (it is the trust boundary). See SECURITY.md.

## DNS
Detect and use, in order: `systemd-resolved` (`resolvectl dns/domain` on the tun interface via bundled up/down script run by the helper with `--script-security 2` **only for that script**), `resolvconf`, NetworkManager. Never run user-supplied `up`/`down`. Setting: "Prevent DNS leaks" (default on). If OpenVPN ≥ 2.6 `--dns` options are present, translate them instead of using legacy `dhcp-option`.

## Concurrency & UI updates
- Backend publishes `AppState` via `tokio::sync::watch`; UI side coalesces to ≤ 60 Hz using `slint::invoke_from_event_loop`.
- Stats tick 1 Hz; globe renders at display rate only while visible and animating, else on-demand.

## Error handling
- Typed `FailReason`: `AuthFailed`, `TlsHandshake`, `Timeout`, `HelperDenied`, `OpenVpnMissing`, `ConfigRejected(reason)`, `NetworkDown`, `Unknown(String)` → each maps to a human message + suggested action in the UI.

## Extensibility hooks (do not build in v1)
Multi-hop, WireGuard backend (`trait Backend`), split tunnelling, server latency probing, provider API import.
