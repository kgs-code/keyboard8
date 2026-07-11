# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

This is a **ZMK firmware user config** for a split **Corne** (`corne_left` / `corne_right`) keyboard running on `nice_nano` controllers. It contains no application source code — only Zephyr/ZMK configuration and device-tree keymap definitions. Firmware is built in CI by ZMK's shared build pipeline, not locally.

## Build & test

There is no local build, lint, or test command. Firmware is compiled entirely in GitHub Actions:

- `.github/workflows/build.yml` delegates to the reusable workflow `zmkfirmware/zmk/.github/workflows/build-user-config.yml@main`.
- Builds trigger on every `push`, `pull_request`, and manual `workflow_dispatch`.
- The build matrix comes from `build.yaml` (note: this is a separate file from the workflow). Each `include` entry is a `board` + `shield` pair; currently `nice_nano//zmk` × `corne_left` and `corne_right`.
- Firmware `.uf2` files are produced as CI artifacts. To flash: download the artifacts, then copy each `.uf2` onto the corresponding half while it is in bootloader mode.

To verify a keymap change, push the branch and check that the GitHub Actions build succeeds — a malformed keymap or `.conf` fails the build. There is no way to validate locally without a full Zephyr/west toolchain.

## Key files

- `config/corne.keymap` — the entire keyboard behavior: device-tree source defining `behaviors` (tap-dances), `combos`, `macros`, and the `keymap` layers. This is where nearly all changes happen.
- `config/corne.conf` — Kconfig feature flags (OLED display, BLE, WPM/battery widgets, TX power, etc.). Uncomment lines to enable optional features like RGB underglow.
- `config/west.yml` — west manifest pinning the ZMK dependency (`zmkfirmware/zmk`, revision `main`). Add external ZMK modules (extra boards/shields/behaviors) as additional `projects` here.
- `build.yaml` — the CI build matrix (which board/shield combos to compile).

## Keymap architecture

The keymap is a Zephyr device-tree overlay (`/ { ... }`). Understanding it requires reading `config/corne.keymap` as a whole, since layers reference behaviors/macros defined above them.

**Physical layout & key positions.** The Corne has 42 keys, indexed 0–41 for combos. Rows are 12 keys wide (positions 0–11, 12–23, 24–35) plus a 6-key thumb cluster (36–41). Each layer's `bindings` block is laid out in this 3×12 + thumb shape; the ASCII comments above `default_layer`, `lower_layer`, and `raise_layer` document the intended legends.

**Layers** (referenced by index in `&mo`, `&to`, `&tog`):
- `0` default_layer — base alpha layer
- `1` layer_1 — near-duplicate of base (toggled via the `WIN` combo; swaps the left thumb from `LCMD` to `LEFT_CONTROL` for Windows vs. Mac)
- `2` lower_layer — numbers, nav arrows, BT output toggle, screenshot/symbol macros
- `3` raise_layer — symbols/punctuation and arrows
- `4` layer_4 — app shortcuts (`LS(LA(...))` chords) and the `symbol_*` unicode-entry macros
- `5` layer_5 — Bluetooth management (`&bt BT_SEL/BT_DISC/BT_CLR`) and volume
- `6` layer_6 — output selection (`OUT_BLE`/`OUT_USB`) and misc
- `7` layer_7 — layer switcher (`&to N`)

**Momentary vs. toggle vs. combos.** Layers are entered momentarily with `&mo N` from thumb keys, and switched persistently through `combos` (chorded key-positions → `&to N` / `&tog N`). The combos section is the main way layers 3/4/5/7 are reached, so changing a `key-positions` set there changes how a layer is accessed. Example combos: positions `38 37 36` → `&to 0` (home), `40 41` → `&out OUT_TOG`.

**Tap-dances** (`tdQ`, `td1`, `tdSL`, `tdDA`, `tdTL`, …) give one key two behaviors by tap count, all with `tapping-term-ms = 180`. Most alpha/number tap-dances map tap → key, double-tap → shifted key (e.g. `&kp Q` / `&kp LS(Q)`).

**Macros.** Two kinds:
- `symbol_*` macros type a Unicode code point followed by `LA(LS(F10))` — this is the trigger for a system/IME hex-to-Unicode input method, so these only produce the intended glyph on a machine configured for that input method.
- Text macros like `mail`, `work`, `symbol_home`, `symbol_1p` type fixed strings (some end with `&to 0` to return to the base layer). `mail` types a personal email address.

### Conventions when editing the keymap

- Preserve the 3×12 + thumb alignment in every layer's `bindings`; the columnar whitespace is intentional and makes the layout readable.
- Keep the ASCII-art legend comments in sync when you change `default_layer`, `lower_layer`, or `raise_layer`.
- Behaviors and macros must be declared in their respective `behaviors {}` / `macros {}` blocks before being referenced with `&name` in a layer.
- Node **labels** (the `foo:` before the brace) are what you reference with `&foo`; the internal names/`label` strings are cosmetic and are sometimes intentionally mismatched (e.g. `symbol_9: symbol_8`). Reference by the label, not the name.
- After any change, the only validation is a green CI build — always let the GitHub Actions workflow run before considering a keymap edit done.
