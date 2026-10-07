# UI_DESIGN.md — Material You, dark

Reference: the Proton VPN screenshot (connected state). Recreate its information architecture with Material 3 styling.

## Principles
- Dark theme only (v1). Tonal surfaces, not shadows. Large rounded shapes. Calm, glanceable.
- One primary action per screen: **Connect / Disconnect**.
- Everything colour-related comes from `Theme` tokens generated from a seed colour (default seed `#2BB673` green-teal; user-changeable with a palette picker).

## Tokens
- **Colour roles:** `surface` (window), `surface_container_low` (sidebar), `surface_container` (cards), `surface_container_high` (hero card, dialogs), `primary` (Connect), `primary_container` (selected row/chips), `tertiary` (origin marker), `secondary` (VPN marker/arc), `error`, custom `success` (Protected badge).
- **Shape:** extra-small 4, small 8, medium 12, large 16, extra-large 28, full (pill) 999 dp. Cards = 16–28, buttons = full, chips = 8, window content = 28 on large containers.
- **Elevation:** use surface tints (tone shifts), shadows only for menus/dialogs.
- **Typography (M3 scale):** Display (hero country) 28/36 medium; Title Large 22; Title Medium 16 medium; Body Medium 14; Label Large 14 medium; Label Small 11. Font: Roboto Flex / Inter fallback, bundled; tabular figures for stats.
- **Spacing:** 4 dp grid; 8/12/16/24 paddings.
- **Motion:** M3 easing (emphasised 500 ms for hero/globe moves, standard 200–300 ms for UI). State-layer overlays: hover 8 %, focus 10 %, pressed 10 %.
- **Icons:** Material Symbols (rounded), SVG, bundled subset.

## Window layout (≥ 960×600, resizable)
```
┌───────────────┬──────────────────────────────────────────────┬────────┐
│ Search        │        [flag] Switzerland                    │ NetShield│  right rail
│ ───────────── │        Zurich · CH#20                        │ Kill sw. │  (navigation
│ ⟳ Recents     │        ( Disconnect )  ← primary pill        │ Auto-rec │   rail, M3)
│ 🌍 Countries  │                                              │ DNS prot │
│ ★ Profiles    │            ╭───────── 3D GLOBE ─────────╮    │ Split t. │
│ chips: All ·  │            │  ◉ origin ~~~arc~~~> ◉ VPN  │    │ Settings │
│ Fav · UDP·TCP │            ╰────────────────────────────╯    │          │
│ ───────────── │                                              │          │
│ ⚡ Fastest     │  ✔ Protected · 2 h 57 min                    │          │
│ 🇦🇱 Albania ▸ │  VPN IP   Server load   Protocol   Volume    │          │
│ 🇩🇿 Algeria ▸ │  Speed ↓102 MB/s ↑81 MB/s   [sparkline]      │          │
│ …             │                                              │          │
└───────────────┴──────────────────────────────────────────────┴────────┘
```
- **Sidebar (320 dp):** M3 search bar; tabs (Recents / Countries / Profiles) as a navigation list; filter chips; virtualised country list (`ListView`) with expandable Country → City → Server rows. Row: flag 24 dp, name, server count badge, trailing "connect" icon button on hover/focus. Selected = `primary_container`. Keyboard: ↑/↓ navigate, → expand, ← collapse, Enter connect.
- **Hero card:** country + city + server label; status chip (Connecting… with indeterminate progress, Connected, Failed); primary button 56 dp tall pill, label toggles Connect/Disconnect/Cancel.
- **Globe canvas:** fills centre, transparent background so surface shows through; subtle vignette. Overlay chips anchored to markers: "You · Kolkata, IN · 103.x.x.x" (blurrable) and "VPN · Zurich, CH".
- **Stats strip (bottom):** 4 columns of label/value (Label Small over Title Medium) + dual-series sparkline (download = primary, upload = tertiary) with axis ticks; values animate with number rolling ≤ 200 ms.
- **Right rail (72 dp):** icon + label toggles; Settings opens a side sheet/dialog. Collapses to a bottom bar < 800 px width.

## Screens & dialogs
1. **First run wizard:** choose `.ovpn` directory → scan summary (N servers, M countries, problems) → geo consent → seed colour → done.
2. **Main** (above).
3. **Profile editor** (modal): name, target (server / city / country / fastest), credentials (stored in keyring), options.
4. **Credentials / 2FA prompt** (modal, M3 dialog): username, password, "Remember", OTP field when challenged.
5. **Settings** (full-height side sheet): General, Appearance, Network (DNS, kill switch), Privacy (geo provider, blur location), Advanced (script policy, openvpn path), About.
6. **Problems panel**: list of unparseable/rejected `.ovpn` files with reasons and "Reveal in file manager".
7. **Log viewer** (P1).
8. **Empty states:** no directory chosen, no servers found, offline geo (illustration + action button).

## Interaction details
- Connect progress: hero ring animates through states; each state label appears under the button.
- On connect: confetti-free; globe eases to frame both markers (500 ms), arc draws origin→VPN (800 ms), "Protected" badge fades in with `success` colour.
- On disconnect: arc retracts, camera eases back, real IP shown with `error`-toned "Not protected" chip.
- Errors: M3 snackbar with action ("Retry", "View log").
- Respect reduced-motion setting everywhere.

## Accessibility
- All icon buttons have `accessible-label`; focus ring = 3 dp `primary` outline; text contrast ≥ 4.5:1 on all tonal surfaces; not colour-only status (icon + text).
- Fully operable by keyboard; `Ctrl+K` focuses search; `Ctrl+Enter` quick connect.
