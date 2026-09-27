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
- `corne.conf`, `build.yaml`, and `west.yml` were left untouched in round 1 —
  the first two are addressed in round 2 below.

## Round 2 — plumbing catch-up + Windows port finished

The first round ported the keymap; this round closed the build/config gaps and
finished a half-done modifier port.

### Windows modifiers (now complete, and a deliberate divergence from keyball)

Round 1 changed only `copy_paste` and the pointer-layer tab keys to `LC`, leaving
the rest Mac. keyball has since **reverted to Mac** (`f24b99f`), so corne is now
the only Windows config and modifiers are an expected permanent diff.

| binding | was (Mac) | now (Windows) |
|---|---|---|
| home-row index | `LEFT_COMMAND F` / `RIGHT_COMMAND J` | `LEFT_CONTROL` / `RIGHT_CONTROL` |
| home-row ring | `LCTRL S` / `RCTRL L` | `LEFT_GUI` / `RIGHT_GUI` |
| bottom-right pinky | `&kp RIGHT_COMMAND` | `&kp RIGHT_CONTROL` |
| `cmd` combo | `LEFT_COMMAND` | `LEFT_CONTROL` (node still named `cmd`) |
| `tab_prev` / `tab_next` combos | `LS(LG([))` / `LS(LG(]))` | `LC(LS(TAB))` / `LC(TAB)` |
| `tab_refresh` combo, pointer refresh | `LG(R)` | `LC(R)` |
| pointer close-tab | `LG(W)` | `LC(W)` |
| pointer screenshot | `LS(LG(N4))` | `LG(LS(S))` (Win+Shift+S — `LG` is correct here) |
| paren next-tab | `LS(LG(]))` | `LC(TAB)` |

GUI/Ctrl were **swapped** rather than both mapped to Ctrl, so the index finger
still carries the primary shortcut modifier the way Cmd did.

### build.yaml

- Added the `settings_reset` build. `flash.md` used to document its absence as a
  hole; that note is gone.
- Added a ZMK Studio build. keyball sets `CONFIG_ZMK_STUDIO=y` in its own
  `keyball44_right.conf`; corne uses the **upstream** shield with no `.conf` of
  ours, so it goes through `cmake-args` instead. It is on **`corne_left`**
  because Studio must run on the central half — keyball's is on the right only
  because the trackball makes the right half central there.

### corne.conf

Ported the portable settings; skipped the trackball-motivated BLE tuning.

- `CONFIG_ZMK_SPLIT_BLE_CENTRAL_BATTERY_LEVEL_FETCHING` / `_PROXY`.
- `CONFIG_ZMK_BEHAVIORS_QUEUE_SIZE=512` — the default 64 can drop events in this
  tap-dance-heavy keymap.

Deliberately **not** ported:

- `CONFIG_BT_PERIPHERAL_PREF_*` (7.5ms interval, `LATENCY=0`), the `BT_BUF_*` /
  `L2CAP_TX_BUF_COUNT` bumps. These exist to kill trackball lag; corne has no
  trackball and they cost battery.
- `CONFIG_ZMK_EXT_POWER=y` — not needed for nice_view's SPI epaper.
- `CONFIG_ZMK_BLE_EXPERIMENTAL_CONN=y` — verified to still exist in v0.3.0
  (`app/Kconfig`), so it is safe to add; simply not adopted. Note the general
  hazard it was first flagged for is real: an undefined `CONFIG_` in a `.conf`
  is silently ignored, so it reads as applied while doing nothing.

### Two settings tried and backed out — read before re-adding

Both were ported from keyball on reasoning the CI build then disproved.

**`CONFIG_ZMK_DISPLAY=y` is redundant.** The upstream `nice_view` shield ships
`app/boards/shields/nice_view/nice_view.conf`, which already sets it *and*
`CONFIG_ZMK_DISPLAY_BLANK_ON_IDLE=n`. The screen was never dead weight: the
pre-change build at `cda18b1` already compiled LVGL, the `ls0xx` driver and
`widgets/status.c`, and adding the line moved flash by only ~1.7 KB. Do not
restate it here — the shield owns it.

**`CONFIG_ZMK_IDLE_TIMEOUT=900000` is a battery regression.** It does not gate
screen blanking on this board (the shield disabled blanking), and the idle state
is not display-only: `app/src/activity.c` → `ZMK_ACTIVITY_IDLE` makes
`display/main.c` call `stop_display_updates()` and `battery.c` stop the battery
poll timer. Raising the timeout from its 30s default to 15m therefore keeps
display refresh and battery polling alive for fifteen minutes of idleness.
corne keeps the default and only sets `IDLE_SLEEP_TIMEOUT=900000`. keyball44
carries the raised value; that is keyball's call, not a gap here.

### .gitignore

Added, with keyball's un-ignore block. The global `~/.gitignore` ignores
`.claude`, so without this any new project command file silently fails to stage
(`flash.md` only got in via `git add -f`).

### flash.md

Ported keyball's macOS permission/retry hardening (`2a70ebf`) so the file stays a
near-1:1 port even though corne flashes over WSL. Also taught the LEFT glob to
exclude the new `corne_left_with_studio` artifact.

### Do not sync west.yml / build.yml

corne is **ahead**: ZMK `v0.3.0` vs keyball's `v0.2`. Sync runs the other way for
these two files.

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
