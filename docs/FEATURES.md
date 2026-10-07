# FEATURES.md — Requirements & Acceptance Criteria

Priority: **P0** must ship in v1, **P1** v1 if time, **P2** later.

## F1. Server directory scan & tree (P0)
- Setting `ovpn_dir` (folder picker; first-run prompt). Recursive scan for `*.ovpn`.
- Each file → `Server { id, path, display_name, country (ISO-3166 alpha-2), city, remote_host, remote_port, proto, tags }`.
- **Country/city resolution, in order:**
  1. Directory layout: `<dir>/<Country>/<City>/<file>.ovpn` or `<dir>/<Country>/<file>.ovpn`.
  2. Filename pattern: `us-nyc-001.ovpn`, `ch-zrh.prod.provider.com_udp.ovpn`, `Switzerland_Zurich_12.ovpn` (ISO code, country name, IATA code, city name).
  3. In-file metadata comments: `# country: CH`, `# city: Zurich` (user-editable override).
  4. Offline GeoIP of the `remote` host (resolved IP) if a DB is configured.
  5. Else `Unknown / Unsorted`.
- User can manually reassign a server's country/city (stored as an override, never edits the file).
- Sidebar shows: **Fastest/Quick connect**, Recents, Favourites, then Countries (flag, name, server count) → expand to Cities → expand to Servers. Search box filters all levels. Filters: protocol (UDP/TCP), favourites.
- Live rescan on directory change (debounced); 5,000 files must scan in < 2 s and never freeze the UI.
- **Accept:** a test directory of 2,000 synthetic files with mixed naming schemes renders the correct tree; unknown files appear under "Unsorted"; malformed files are listed in a "Problems" panel with a reason, not silently dropped.

## F2. Saved connections / profiles (P0)
- A **Profile** = server reference (or a country/city "smart" target) + credentials ref + options (preferred proto, auto-reconnect, custom name, favourite flag).
- Create, rename, duplicate, delete, reorder. Persisted in `profiles.toml`.
- Credentials: per-profile or shared "default provider" credentials in the OS keyring. Servers whose `.ovpn` embeds certs/keys and needs no login connect without prompts.
- Handle `auth-user-pass` (prompt once, offer "remember"), `--askpass`/private-key passphrase, and OTP/2FA challenge (`CRV1` / dynamic challenge) with an inline dialog.
- **Accept:** user saves 3 profiles, restarts the app, all 3 present; passwords are only in the keyring (verify by grepping config files).

## F3. One-click connect (P0)
- Big primary button on the hero card: Connect / Disconnect (label reflects state). Quick Connect picks: last used profile → else fastest (lowest measured latency among favourites/country).
- Clicking any server/city/country row connects (single click on a "Connect" icon button; double-click on the row also connects). Switching servers while connected performs disconnect → connect seamlessly.
- States shown with text and colour: Disconnected, Connecting, Authenticating, Getting config, Assigning IP, Connected, Reconnecting, Failed (with human reason + "View log").
- Auto-reconnect with exponential backoff (setting, default on). Cancel button during connect.
- **Accept:** from a cold start, one click on a profile yields Connected with no extra dialogs (given saved creds); disconnect terminates the openvpn process and removes routes.

## F4. Traffic statistics (P0)
- Live **download** and **upload** speed (e.g. `102 MB/s`, auto-scale B/KB/MB/GB, bits/bytes toggle), **session volume** totals, **60 s sparkline** graph with two series, smoothed (EMA).
- Source: management `bytecount` (1 s). Speed = Δbytes/Δt using monotonic time.
- Lifetime totals + per-server history in `history.json`.
- **Accept:** against the fake management server, a scripted transcript produces exactly the expected speeds/totals; UI updates ≥1 Hz without dropped frames.

## F5. Session timer (P0)
- Starts at the `CONNECTED` state timestamp (not at click), shows `2 h 57 min` / `HH:MM:SS` (setting). Survives UI restarts if the VPN is still up (re-attach to running session via management socket).
- Resets on disconnect; reconnects start a new session but optionally show "total since first connect".
- **Accept:** timer matches wall-clock within 1 s over a 1-hour test; app restart while connected resumes the timer correctly.

## F6. 3D globe with real vs VPN location (P0)
- Textured dark globe, auto-rotating when idle, drag to rotate, scroll/pinch to zoom, double-click to recenter.
- **Origin marker (real location):** fetched via IP geolocation **before** the tunnel comes up and cached; shown with a pulsing ring. Hidden/blurred if the user enabled "hide my real location".
- **VPN marker:** after CONNECTED, fetch exit IP/location **through the tunnel** (verify it differs from origin); marker with glow.
- **Arc:** great-circle arc (slerp, raised above the surface) animating from origin to VPN exit on connect; camera eases to frame both points; arc fades on disconnect and camera returns.
- Cards over the globe: country flag + name, city, VPN exit IP and the internal tunnel IP; "Protected" badge when connected, "Not protected" with real IP when not.
- Geo provider failure → degrade gracefully (use server's resolved country centroid; show "approximate").
- Reduced-motion setting disables auto-rotate and arc animation.
- No GPU / wgpu failure → automatic 2D map fallback with the same markers and arc.
- **Accept:** golden-image snapshot tests for camera framing math; arc endpoints exactly match marker lat/lon; 60 fps on integrated GPU at 1080p.

## F7. Layout parity with reference screenshot (P0)
- Left: search + nav (Recents, Countries, Profiles) + chips (All, Secure Core→"Favourites", P2P→"UDP", Tor→"TCP") + country list with flags.
- Centre: connection hero (country flag + name, city, server id, Connect/Disconnect button) over the globe.
- Right rail: quick toggles (Kill switch, Auto-reconnect, DNS leak protection, Split tunnelling placeholder, Settings).
- Bottom strip: status "Protected", session time, VPN IP, server load (if available from filename/metadata, else hidden), protocol, volume, download/upload speed + sparkline.

## F8. Settings (P0/P1)
- P0: `.ovpn` directory, theme seed colour, connect-on-launch, auto-reconnect, units, geo provider, start minimised to tray.
- P1: custom DNS, kill switch (nftables, helper-applied), launch on login, notifications, log viewer, import/export profiles, offline GeoIP DB path.
- P2: split tunnelling, ping-based "fastest server" background probing, multi-hop, Windows/macOS.

## F9. Tray & notifications (P1)
- Tray icon with status colour; menu: Connect/Disconnect, Quick Connect, recent profiles, Show/Quit. Toasts on connect, disconnect, failure.

## F10. Logs & diagnostics (P1)
- In-app log viewer (redacted), "Copy diagnostics" (versions, OS, openvpn version, last 200 redacted lines).

## Non-functional
- Cold start < 1 s to window; idle CPU < 1 %, idle RAM < 150 MB (with globe); globe pauses rendering when window hidden.
- Offline-first: app works with no internet except geo lookups.
- Accessibility: keyboard nav, focus rings, contrast ≥ 4.5:1 for text, labels on icon buttons.
- i18n-ready strings (`@tr()` in Slint), English first.
