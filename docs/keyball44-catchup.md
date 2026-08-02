# Corne ← Keyball44 catch-up

Context for the layout convergence landed in `feat/catchup-keyball44-mirror`
(commit `7511002`).

## Why

This corne config was the original; **keyball44**
(`bradfordwagner/zmk-config-Keyball44`) was forked from it. Since the fork,
keyball44 became the daily driver and its keymap iterated well ahead of corne.
This change brings corne back into sync by making it a **structural mirror** of
keyball44, so every future catch-up is a near-1:1 port instead of a translation.

Direction of sync: **keyball44 → corne**.

## The two structural facts that make this a semantic port (not a file copy)

1. **Layer numbers had diverged.** Same typing layers, different indices. corne
   was renumbered to match keyball exactly:

   | Purpose            | old corne idx      | new corne idx (= keyball) |
   |--------------------|--------------------|---------------------------|
   | default / arrows / numpad / g1 | 0 / 1 / 2 / 3 | 0 / 1 / 2 / 3      |
   | g2                 | 4                  | **8**                     |
   | paren              | 6                  | **7**                     |
   | pointer stack      | ploopy 5, mouse 7, mouse2 8, mouse3 9 | MOUSE 4 / SCROLL 5 / SNIPE 6 |
   | STS (game)         | —                  | **9**                     |

2. **Different pointing hardware.** keyball44 has an onboard PMW3610 trackball
   (automouse/scroll/snipe layer activation, CPI tuning, sensor conf). corne
   drives an external ploopy + keyboard-driven `mmv`/`msc` mouse emulation. The
   pointer half is therefore hardware-specific and is **not** blind-ported.

## Thumb-row / combo position mapping

corne is a 42-key 3+3 thumb layout (positions 36–41). keyball pads its outer
thumb cluster with two `&none`, putting the same logical thumbs at 38–43.

- **Ported thumb row** = keyball's row **minus the two leading `&none`**.
- **Combo thumb position** = keyball's position **− 2** (e.g. keyball `40 → 38`,
  `39 → 37`). Positions in the top three rows (0–35) are unchanged.
- Layer-lists inside combos are copied verbatim, since the numbering now matches.

## What was ported

- **Layer renumber** to match keyball (table above); combo `layers` and thumb
  `key-positions` updated accordingly.
- **Home-row mods:** `home_row` → `tap-preferred`, tapping-term 150,
  quick-tap 150, require-prior-idle 100.
- **Global behavior tuning:** `&lt` (240 / balanced / quick-tap 200) and `&mt`
  (200 / tap-preferred / quick-tap 150).
- **`caps_word`** continue-list = `<UNDERSCORE MINUS>`.
- **numpad layer:** `PG_UP`/`PG_DN` on the `;` / `/` columns; `dot_comma` moved
  to the top-right.
- **default layer:** numpad moved from hold-`C` → hold-`Backspace`
  (`&lt 2 BACKSPACE`), freeing `C` to a plain key; `MINUS` thumb → `&lt 7`
  (paren, renumbered).
- **Windows bindings:** `copy_paste` → `LC(C)` / `LC(V)`; pointer layers carry
  keyball's `LC(TAB)` / `LC(LS(TAB))` tab-switch keys.
- **Pointer layers:** corne's `ploopy` / `mouse` / `mouse2` / `mouse3` collapsed
  into keyball's `MOUSE` / `SCROLL` / `SNIPE`. Bindings are keyboard-driven
  (`mmv` / `msc` / `mkp`), reached via the `to_mouse` / `to_scroll` combos and
  `&lt 6 ESCAPE`, exactly as keyball does.
- **STS layer** (layer 9) plus its `td_sts_end` and `mo_mkp` behaviors.

## What was intentionally skipped (hardware / divergent)

- PMW3610 conf and `_left`/`_right` overlays (automouse/scroll/snipe, CPI).
- The `&nice_view_spi { cs-gpios … }` line (keyball shield wiring).
- The `#include <behaviors/mouse_keys.dtsi>` (not needed on corne's ZMK pin;
  corne resolves `mkp`/`mmv`/`msc` via `behaviors.dtsi` + `pointing.h`).
- keyball's BLE trackball-lag tuning in `*.conf`.
- `corne.conf`, `build.yaml`, and `west.yml` were left untouched.

## Known caveat — SNIPE is click-only on corne

On keyball, slow-precision pointing is the SNIPE layer **dropping the sensor
CPI**. corne cannot change ploopy's CPI from firmware, so its SNIPE layer is
currently just the click bindings (mostly `&trans`). corne's old
`mouse2`/`mouse3` slow-`mmv` layers were retired in the collapse. If
keyboard-driven slow movement is wanted on corne, add `&mmv MOVE_X(1200)`-style
keys into the SNIPE layer to restore it.

## Verification status

- **CI build: passing.** Run
  [`30727031158`](https://github.com/bradfordwagner/zmk-config-corne/actions/runs/30727031158)
  built both halves (`corne_left` / `corne_right`) and produced the `firmware`
  artifact. This proves the keymap **compiles and flashes**.
- **Not yet verified on hardware:** feel and layout — the tap-preferred HRM
  timing, numpad-on-hold-Backspace, the Windows `LC` bindings, and the
  SNIPE-is-click-only behavior. Flash both halves (grab the `firmware` artifact
  or use `/flash`) and do a real typing pass before trusting it.
