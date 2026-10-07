# TESTING.md

## Pyramid
1. **Unit** (fast, pure): parser, resolver, geo math, stats engine, state machine, theme tokens.
2. **Property** (`proptest`): filenames → resolver never panics and is idempotent; arbitrary bytes → parser returns Ok/Err, never panics; slerp arc endpoints equal inputs; stats monotonic totals.
3. **Integration**: `orbit-mgmt` against a **fake OpenVPN management server** replaying transcripts (`tests/transcripts/*.txt`): happy path, AUTH_FAILED, TLS error, mid-session reconnect, `CRV1` challenge, garbage lines, socket drop.
4. **Session tests** with `MockBackend`: full connect/disconnect/backoff using `tokio::time::pause()` (virtual time) — no real sleeping.
5. **UI tests**: Slint headless backend (`i-slint-backend-testing`) for navigation, keyboard, state→view mapping; snapshot tests (`insta`) for view-model output.
6. **Globe**: unit-test camera framing & projection; render one offscreen frame in CI (llvmpipe/lavapipe software Vulkan) and compare to a golden PNG with tolerance.
7. **Fuzz**: `cargo fuzz` targets for `.ovpn` parser and management line decoder (nightly CI job).
8. **Manual/E2E checklist** (before each release; needs real openvpn + a test server, e.g. a local OpenVPN container): connect, stats move, timer, kill, reconnect, DNS check, kill switch, tray, restart-while-connected.

## Test data (`testdata/`, synthetic only)
- `ovpn/mixed_2000/` generator script producing nested dirs and flat files with 6 naming schemes, unicode names, duplicates, broken files, symlink loops, a 5 MB junk file.
- `ovpn/malicious/` files with `up`, `plugin`, `script-security 3`, include loops — must be rejected.
- `mgmt/*.txt` transcripts.

## Performance budgets (CI benchmarks with `criterion`)
- Scan 5,000 files < 2 s; resolver 5,000 names < 50 ms; stats tick < 100 µs; globe frame < 6 ms on integrated GPU.

## CI jobs
`fmt` · `clippy -D warnings` · `test` (ubuntu, + windows/macos compile-check) · `deny` · `audit` · `fuzz-smoke (60 s)` · `build-release` artifact.

## Coverage target
≥ 80 % lines in `orbit-core`, `orbit-ovpn`, `orbit-mgmt`, `orbit-session`; helper validation branch coverage 100 % on reject paths.
