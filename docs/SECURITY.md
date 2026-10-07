# SECURITY.md

## Assets
User traffic privacy, VPN credentials, private keys inside `.ovpn` files, the user's real IP/location, the user's machine (root via helper).

## Threat model (what we defend against)
1. **Malicious `.ovpn`** (downloaded from a provider or third party) executing code via `up`/`down`/`plugin`/`script-security`.
2. **Local unprivileged attacker** abusing the privileged helper (path traversal, symlink races, arbitrary argv).
3. **Credential leakage** via logs, crash dumps, config files, process args, environment.
4. **Leaks while "connected"**: DNS, IPv6, WebRTC-like leaks, tunnel not actually routing.
5. **Geo-IP lookups** revealing the real IP to third parties without consent.
6. **Supply chain** (malicious/abandoned crates).

## Privilege model
- `orbit-app` runs as the user. `orbit-helper` is the only root component, launched per-connection via polkit (`pkexec` + a dedicated polkit action `org.orbit.vpn.run` with `auth_self_keep` or admin-configurable policy). No long-running root daemon in v1.
- Helper inputs are **untrusted**. It must:
  - `canonicalize()` the config path and require it to be inside an allow-listed directory recorded at install/first-run (re-checked, no symlink escape, regular file, not world-writable, owned by the invoking uid or root).
  - Re-parse the config itself and apply the directive policy below (do not trust the GUI's verdict).
  - Build the `openvpn` argv itself from an allow-list; reject unknown flags.
  - Create the management socket in a root-owned dir it then `chown`s to the invoking uid, mode 0600; or accept only a path under `/run/user/<uid>/orbit/` verified to be owned by that uid.
  - Run openvpn with minimal env, `--script-security 1` (or 2 only for the bundled DNS script), `--user nobody --group nogroup` where compatible (`--persist-tun --persist-key`), `--auth-nocache`.
  - Drop capabilities not needed (keep `CAP_NET_ADMIN`, `CAP_NET_RAW`, `CAP_NET_BIND_SERVICE` if required).

## `.ovpn` directive policy
| Class | Directives | Default |
|---|---|---|
| Script/exec | `up`, `down`, `route-up`, `route-pre-down`, `ipchange`, `tls-verify`, `auth-user-pass-verify`, `client-connect`, `client-disconnect`, `learn-address`, `plugin`, `setenv`/`setenv-safe` (env tricks), `script-security >1`, `management*` (we set our own), `config` (include) outside dir | **Rejected** (profile shows "needs review"; per-profile explicit opt-in with a warning dialog) |
| Network-sensitive | `redirect-gateway`, `route`, `dhcp-option`, `block-outside-dns` | Allowed (expected) |
| TLS safety | `tls-version-min`, `verify-x509-name`, `remote-cert-tls` | Allowed; warn if `verify-x509-name`/`remote-cert-tls` missing |
| Insecure | `cipher none`, `auth none`, `comp-lzo`/`compress` without `--allow-compression no` | Warn; reject `cipher none`/`auth none` |
Unknown directives pass through but are logged at `debug`.

## Secrets
- Keyring only (`keyring`, `secrecy::SecretString`, `zeroize`). Passed to openvpn **via the management interface** (`username`/`password` commands), never on argv, env, or temp files.
- Embedded private keys in `.ovpn` are read by openvpn only; GUI never displays or logs them. Files with encrypted keys prompt for passphrase via management `>PASSWORD:Need 'Private Key'`.
- Log redaction layer: strip IPs of the user's real address at `info`+, strip anything matching `<auth-user-pass>`, `<key>…</key>`, `password`, `token`.
- Core dumps disabled for the helper (`prctl(PR_SET_DUMPABLE, 0)`); mlock secrets where feasible.

## Leak protection
- DNS: applied via the helper (see ARCHITECTURE §DNS); verified post-connect with a lookup through the tunnel resolver.
- IPv6: if the server does not push v6, block v6 (`--block-ipv6` equivalent via helper nft rule) when "Prevent leaks" is on.
- **Kill switch (P1):** helper installs an nftables ruleset in its own table `inet orbit_killswitch` allowing only loopback, the tunnel interface, and the VPN server endpoint; removed on intentional disconnect; crash-safe (rules persist until app/helper cleans them, with a "Restore normal networking" button).
- Exit-IP check: after CONNECTED, compare exit location/IP with origin; warn if identical.

## Network calls the app may make
Only: (1) geo-IP providers (HTTPS, rustls, certificate verification **always on**, user-consented, disable-able), (2) optional update check (off by default). **No telemetry.** Geo calls for the *origin* are made before connecting; for the *exit* through the tunnel.

## Supply chain & CI
`cargo deny` (licences, bans, advisories), `cargo audit` weekly, Dependabot/Renovate, `Cargo.lock` committed, reproducible release builds, minimal feature flags (no OpenSSL where rustls works).

## Reporting
`SECURITY` contact placeholder in README; coordinated disclosure 90 days.

## Review checklist for any PR touching `orbit-helper`, `orbit-session` spawn logic, or `.ovpn` parsing
- [ ] Every new argv element allow-listed and validated
- [ ] No TOCTOU between check and use (use `openat`/fd passing where possible)
- [ ] Errors do not leak secrets
- [ ] Fuzz corpus updated
- [ ] Threat model table still accurate
