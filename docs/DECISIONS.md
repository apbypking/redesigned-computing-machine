# DECISIONS.md — Architecture Decision Records (append-only)

Format: `## ADR-NNN: Title — YYYY-MM-DD — Status` then Context / Decision / Consequences.

## ADR-001: Slint as GUI toolkit — Proposed
- **Context:** Need Rust-native Material 3 dark UI with GPU 3D integration.
- **Decision:** Slint with built-in Material style; custom M3 components where missing; wgpu globe rendered to texture.
- **Consequences:** Verify wgpu-texture import feature and licence terms against the current Slint release at M0; fallback is Iced (see TECH_STACK.md). Record pinned version here.

## ADR-002: Control OpenVPN via management interface — Accepted
- Authoritative state/bytecount; credentials never on argv; clean shutdown.

## ADR-003: Privileged helper per connection via polkit — Accepted
- No long-running root daemon in v1; revisit for kill switch persistence.

## ADR-004: Linux-first — Accepted
- Primary: Linux (Wayland/Hyprland, X11). Keep Windows/macOS ports possible via `trait Backend` and abstracted tray/keyring/DNS.
