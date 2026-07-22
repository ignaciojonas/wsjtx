# Colorblind-Accessible Decode Highlighting Palettes

## Problem

WSJT-X's "Decode Highlighting" feature (Configuration → Colors tab) colors decoded
text by category (new DXCC, new grid, new callsign, CQ, own transmit, LotW user,
etc.), using two hardcoded presets ("Default 1" and "Default 2") plus per-item
manual color pickers. Neither preset was designed with color vision deficiency in
mind, making it hard for colorblind or low-vision users to distinguish categories.

## Goal

Add three additional predefined, colorblind/low-vision-friendly presets for decode
text colors, applied the same way the existing "Default 1"/"Default 2" presets are,
without touching persistence, the highlighting engine, or manual per-item editing.

Out of scope: waterfall palette (already has its own preset system via `.pal`
files), user-savable/named custom schemes, anything beyond decoded-text
highlighting colors.

## Current mechanism (for reference)

- `models/DecodeHighlightingModel.hpp/.cpp` holds 16 `Highlight` enum values (CQ,
  MyCall, Tx, DXCC, DXCCBand, Grid, GridBand, Call, CallBand, Continent,
  ContinentBand, CQZone, CQZoneBand, ITUZone, ITUZoneBand, LotW), each with an
  enabled flag, foreground `QBrush`, and background `QBrush`.
- Two static preset arrays already exist: `impl::defaults_` ("Default 1") and
  `impl::defaults2_` ("Default 2"), exposed via `default_items()` /
  `default_items2()`.
- `Configuration.ui` (Colors tab, `groupBox_12` / `horizontalLayout_20`) has two
  `QPushButton`s wired in `Configuration.cpp` (`on_reset_highlighting_to_defaults_push_button_clicked`,
  `on_reset_highlighting_to_defaults2_push_button_clicked`) that, after a
  `MessageBox::query_message` confirmation, replace
  `next_decode_highlighing_model_`'s items with the chosen preset.
- On dialog accept, `Configuration.cpp:3591-3595` diffs
  `next_decode_highlighing_model_.items()` against the live model, applies it, and
  `write_settings()` persists the whole `HighlightItems` list under the single
  `QSettings` key `"DecodeHighlighting"` (custom `QDataStream` serialization).
- `widgets/displaytext.cpp` (`set_colours()`) reads the model directly and applies
  colors to decoded text — it is agnostic to where the colors came from.

## Design

### UI

Add one new `QPushButton` ("Apply Accessibility Palette ▾") to the existing button
row in `Configuration.ui` (`horizontalLayout_20`), next to "Reset Highlighting to
Default 1/2" and "Rescan ADIF Log". It carries a `QMenu` with three actions:

- "Red-Green safe (Deuteranopia/Protanopia)"
- "Blue-Yellow safe (Tritanopia)"
- "High contrast"

Chosen over alternatives:
- *A combobox replacing the two existing buttons* — cleaner long-term, but
  rewrites a working pattern and UI for no functional gain.
- *Three more individual buttons* — matches existing pattern exactly but makes
  the button row overflow (6 buttons).

The single button + dropdown menu adds one widget, reuses the existing button row,
and leaves "Default 1"/"Default 2"/"Rescan ADIF Log" untouched.

Each menu action triggers the same confirm-then-apply flow as the existing reset
buttons: a `MessageBox::query_message` confirmation ("Reset all decode highlighting
and priorities to <preset name> values"), then on Yes,
`next_decode_highlighing_model_.items (DecodeHighlightingModel::default_items_X ())`.

### Data model

Add three new static const `HighlightItems` arrays to
`DecodeHighlightingModel::impl` (`deuteranopia_`, `tritanopia_`,
`high_contrast_`), following the exact same enabled-flags-per-type as
`defaults_` (i.e., same items enabled/disabled by default — only the colors
differ), plus three new static accessors mirroring `default_items()` /
`default_items2()`:

```cpp
static HighlightItems const& default_items_deuteranopia ();
static HighlightItems const& default_items_tritanopia ();
static HighlightItems const& default_items_high_contrast ();
```

No changes to `HighlightInfo`, serialization operators, persistence format, or
`displaytext.cpp` — these presets are just alternative seed data for the same
model.

### Color strategy

Foreground text is chosen per-item for max contrast against that item's
background (black or white, whichever contrasts more), not fixed.

1. **Red-Green safe (Deuteranopia/Protanopia)** — backgrounds drawn from the
   Okabe-Ito categorical palette (orange, sky blue, bluish green, yellow, blue,
   vermillion, reddish purple), which is the standard reference palette for CVD
   accessibility. Avoids using pure red vs. pure green as the sole differentiator
   between any two categories.
2. **Blue-Yellow safe (Tritanopia)** — backgrounds drawn from the red/green/
   orange/purple axis (red, green, orange, purple, pink, brown, maroon), which
   tritanopes perceive normally. Avoids relying on blue vs. yellow to
   differentiate categories.
3. **High contrast** — strongly saturated, high-luminance-difference colors;
   "Band" variants (e.g. `ContinentBand`) use a darker/desaturated variant of the
   base category's hue rather than a pale tint (pale tints wash out for low
   vision), preserving the existing semantic pattern that "Band" categories are
   subordinate to their base category.

All three presets follow the existing convention that "Band" variants stay
visually related to (but less prominent than) their base category. Exact hex
values are chosen during implementation and validated with a contrast checker and
a colorblind simulator (e.g. Coblis) before finalizing — not fixed in this spec.

### Persistence & compatibility

No format or settings-key changes. Applying a new preset just replaces the
in-memory `HighlightItems` list the same way "Default 1"/"Default 2" already do;
it is serialized/deserialized through the existing `QDataStream` operators.
Canceling the Configuration dialog after applying a preset discards it exactly
like the existing reset buttons (reverts via `initialize_models()`).

### Documentation

Update `doc/user_guide/en/settings-colors.adoc` to mention the new accessibility
presets alongside the existing Default 1/2 description.

## Testing / verification

No existing automated test coverage for decode-highlighting colors (it's a
QSettings-backed UI model). Verification is manual:

- Apply each of the 3 new presets via the UI, confirm the list view updates and
  colors read correctly.
- Save, restart, confirm the applied colors persist (reuses existing
  persistence path, low risk).
- Cancel after applying a preset, confirm it reverts to the previously saved
  scheme (reuses existing reject path).
- Check each preset's color choices with a colorblind simulator for the
  corresponding vision type before finalizing hex values.
