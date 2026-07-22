# Colorblind-Accessible Decode Highlighting Palettes Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add three predefined, colorblind/low-vision-friendly color presets ("Red-Green safe", "Blue-Yellow safe", "High contrast") for decode text highlighting, applied the same way the existing "Default 1"/"Default 2" presets are.

**Architecture:** Three new static `HighlightItems` arrays in `DecodeHighlightingModel`, exposed via three new static accessor functions, applied through a new "Apply Accessibility Palette ▾" button + `QMenu` in the Configuration dialog's Colors tab, reusing the existing confirm-then-replace-model flow that "Reset Highlighting to Default 1/2" already use. No changes to persistence, serialization, or the highlighting engine (`widgets/displaytext.cpp`).

**Tech Stack:** C++11, Qt5 (Widgets, Gui, Core), CMake, Qt Designer `.ui` XML.

## Global Constraints

- No new `QSettings` keys or serialization changes — reuse the existing `"DecodeHighlighting"` key and `QDataStream` operators for `DecodeHighlightingModel::HighlightInfo`.
- No changes to `widgets/displaytext.cpp` or any other consumer of `DecodeHighlightingModel` — it must remain agnostic to where highlight colors came from.
- Every new preset must have exactly one entry per `DecodeHighlightingModel::Highlight` enum value (16 values: `CQ, MyCall, Tx, DXCC, DXCCBand, Grid, GridBand, Call, CallBand, Continent, ContinentBand, CQZone, CQZoneBand, ITUZone, ITUZoneBand, LotW`), matching the enabled/disabled defaults of `impl::defaults_` (`ITUZone`, `ITUZoneBand`, `Grid`, `GridBand`, `Call`, `CallBand`, `LotW` disabled; all others enabled).
- No new automated test target — per the approved design spec (`docs/superpowers/specs/2026-07-22-colorblind-accessible-highlighting-design.md`), this repo has no unit-test coverage for `QSettings`-backed UI color models, and verification here is a build check plus a manual UI walkthrough.
- Follow existing code style in each file exactly (brace placement, `_` suffix on private members, `tr ()` spacing).

---

### Task 1: Add three accessibility color presets to `DecodeHighlightingModel`

**Files:**
- Modify: `models/DecodeHighlightingModel.hpp:41-42`
- Modify: `models/DecodeHighlightingModel.cpp:28-29` (member declarations), `:34-70` (data), `:152-160` (accessors)

**Interfaces:**
- Produces: `DecodeHighlightingModel::default_items_deuteranopia ()`, `DecodeHighlightingModel::default_items_tritanopia ()`, `DecodeHighlightingModel::default_items_high_contrast ()` — each `static HighlightItems const&`, same shape as the existing `default_items ()` / `default_items2 ()`. Task 2 calls these directly.

- [ ] **Step 1: Declare the three new accessors in the header**

In `models/DecodeHighlightingModel.hpp`, find:

```cpp
  // access to raw items nd default items
  static HighlightItems const& default_items ();
  static HighlightItems const& default_items2 ();
```

Replace with:

```cpp
  // access to raw items nd default items
  static HighlightItems const& default_items ();
  static HighlightItems const& default_items2 ();
  static HighlightItems const& default_items_deuteranopia ();
  static HighlightItems const& default_items_tritanopia ();
  static HighlightItems const& default_items_high_contrast ();
```

- [ ] **Step 2: Declare the three new backing arrays on `impl`**

In `models/DecodeHighlightingModel.cpp`, find:

```cpp
  HighlightItems static const defaults_;
  HighlightItems static const defaults2_;
  HighlightItems data_;
  QFont font_;
```

Replace with:

```cpp
  HighlightItems static const defaults_;
  HighlightItems static const defaults2_;
  HighlightItems static const deuteranopia_;
  HighlightItems static const tritanopia_;
  HighlightItems static const high_contrast_;
  HighlightItems data_;
  QFont font_;
```

- [ ] **Step 3: Add the three preset data arrays**

In `models/DecodeHighlightingModel.cpp`, find the end of `defaults2_` (the line reading `};` right after the `Tx` entry of `defaults2_`, i.e. right before `bool operator == (...)`):

```cpp
  , {Highlight::Tx, true, {Qt::black}, {{0xff, 0xa5, 0xc6}}}
};

bool operator == (DecodeHighlightingModel::HighlightInfo const& lhs, DecodeHighlightingModel::HighlightInfo const& rhs)
```

Replace with (inserting the three new arrays between them):

```cpp
  , {Highlight::Tx, true, {Qt::black}, {{0xff, 0xa5, 0xc6}}}
};

// Backgrounds drawn from the Okabe-Ito categorical palette, the
// standard reference palette for deuteranopia/protanopia
// accessibility. Foreground is black or white, whichever contrasts
// more with that item's background.
QList<DecodeHighlightingModel::HighlightInfo> const DecodeHighlightingModel::impl::deuteranopia_ = {
  {Highlight::MyCall, true, {Qt::white}, {{0xd5, 0x5e, 0x00}}}
  , {Highlight::Continent, true, {Qt::white}, {{0x00, 0x72, 0xb2}}}
  , {Highlight::ContinentBand, true, {Qt::black}, {{0xa8, 0xcc, 0xe3}}}
  , {Highlight::CQZone, true, {Qt::black}, {{0xe6, 0x9f, 0x00}}}
  , {Highlight::CQZoneBand, true, {Qt::black}, {{0xf5, 0xd9, 0x99}}}
  , {Highlight::ITUZone, false, {Qt::black}, {{0x56, 0xb4, 0xe9}}}
  , {Highlight::ITUZoneBand, false, {Qt::black}, {{0xbe, 0xe3, 0xf5}}}
  , {Highlight::DXCC, true, {Qt::black}, {{0xcc, 0x79, 0xa7}}}
  , {Highlight::DXCCBand, true, {Qt::black}, {{0xe8, 0xbf, 0xd6}}}
  , {Highlight::Grid, false, {Qt::white}, {{0x00, 0x00, 0x00}}}
  , {Highlight::GridBand, false, {Qt::black}, {{0xcc, 0xcc, 0xcc}}}
  , {Highlight::Call, false, {Qt::black}, {{0x99, 0x99, 0x99}}}
  , {Highlight::CallBand, false, {Qt::black}, {{0xdd, 0xdd, 0xdd}}}
  , {Highlight::LotW, false, {{0x00, 0x55, 0x44}}, {}}
  , {Highlight::CQ, true, {Qt::white}, {{0x00, 0x9e, 0x73}}}
  , {Highlight::Tx, true, {Qt::black}, {{0xf0, 0xe4, 0x42}}}
};

// Backgrounds drawn from the red/green/orange/purple axis, which
// tritanopia does not impair, avoiding blue vs. yellow as a
// differentiator between categories.
QList<DecodeHighlightingModel::HighlightInfo> const DecodeHighlightingModel::impl::tritanopia_ = {
  {Highlight::MyCall, true, {Qt::white}, {{0xe4, 0x1a, 0x1c}}}
  , {Highlight::Continent, true, {Qt::black}, {{0xf7, 0x81, 0xbf}}}
  , {Highlight::ContinentBand, true, {Qt::black}, {{0xfc, 0xe0, 0xef}}}
  , {Highlight::CQZone, true, {Qt::white}, {{0xa6, 0x56, 0x28}}}
  , {Highlight::CQZoneBand, true, {Qt::black}, {{0xe3, 0xc6, 0xb3}}}
  , {Highlight::ITUZone, false, {Qt::white}, {{0xb0, 0x30, 0x60}}}
  , {Highlight::ITUZoneBand, false, {Qt::black}, {{0xe8, 0xb9, 0xc7}}}
  , {Highlight::DXCC, true, {Qt::white}, {{0x98, 0x4e, 0xa3}}}
  , {Highlight::DXCCBand, true, {Qt::black}, {{0xe0, 0xc6, 0xe6}}}
  , {Highlight::Grid, false, {Qt::black}, {{0x99, 0x99, 0x99}}}
  , {Highlight::GridBand, false, {Qt::black}, {{0xdd, 0xdd, 0xdd}}}
  , {Highlight::Call, false, {Qt::black}, {{0xfd, 0xbf, 0x6f}}}
  , {Highlight::CallBand, false, {Qt::black}, {{0xfd, 0xe9, 0xd0}}}
  , {Highlight::LotW, false, {{0x6e, 0x14, 0x23}}, {}}
  , {Highlight::CQ, true, {Qt::black}, {{0x4d, 0xaf, 0x4a}}}
  , {Highlight::Tx, true, {Qt::black}, {{0xff, 0x7f, 0x00}}}
};

// Strongly saturated, high-luminance-difference colors. "Band"
// variants use a darker/desaturated shade of the base category's
// hue rather than a pale tint, since pale tints wash out for low
// vision.
QList<DecodeHighlightingModel::HighlightInfo> const DecodeHighlightingModel::impl::high_contrast_ = {
  {Highlight::MyCall, true, {Qt::white}, {{0xff, 0x00, 0x00}}}
  , {Highlight::Continent, true, {Qt::black}, {{0xff, 0x80, 0x00}}}
  , {Highlight::ContinentBand, true, {Qt::white}, {{0x99, 0x50, 0x00}}}
  , {Highlight::CQZone, true, {Qt::white}, {{0x00, 0x40, 0xff}}}
  , {Highlight::CQZoneBand, true, {Qt::white}, {{0x00, 0x29, 0x66}}}
  , {Highlight::ITUZone, false, {Qt::white}, {{0x00, 0x80, 0x80}}}
  , {Highlight::ITUZoneBand, false, {Qt::white}, {{0x00, 0x4d, 0x4d}}}
  , {Highlight::DXCC, true, {Qt::white}, {{0xcc, 0x00, 0xcc}}}
  , {Highlight::DXCCBand, true, {Qt::white}, {{0x7a, 0x00, 0x7a}}}
  , {Highlight::Grid, false, {Qt::white}, {{0x66, 0x66, 0x66}}}
  , {Highlight::GridBand, false, {Qt::white}, {{0x33, 0x33, 0x33}}}
  , {Highlight::Call, false, {Qt::white}, {{0xd6, 0x00, 0x6d}}}
  , {Highlight::CallBand, false, {Qt::white}, {{0x80, 0x00, 0x41}}}
  , {Highlight::LotW, false, {{0x00, 0x1a, 0x66}}, {}}
  , {Highlight::CQ, true, {Qt::white}, {{0x00, 0xa0, 0x00}}}
  , {Highlight::Tx, true, {Qt::black}, {{0xff, 0xff, 0x00}}}
};

bool operator == (DecodeHighlightingModel::HighlightInfo const& lhs, DecodeHighlightingModel::HighlightInfo const& rhs)
```

- [ ] **Step 4: Add the three new accessor function definitions**

In `models/DecodeHighlightingModel.cpp`, find:

```cpp
auto DecodeHighlightingModel::default_items2 () -> HighlightItems const&
{
  return impl::defaults2_;
}
```

Replace with:

```cpp
auto DecodeHighlightingModel::default_items2 () -> HighlightItems const&
{
  return impl::defaults2_;
}

auto DecodeHighlightingModel::default_items_deuteranopia () -> HighlightItems const&
{
  return impl::deuteranopia_;
}

auto DecodeHighlightingModel::default_items_tritanopia () -> HighlightItems const&
{
  return impl::tritanopia_;
}

auto DecodeHighlightingModel::default_items_high_contrast () -> HighlightItems const&
{
  return impl::high_contrast_;
}
```

- [ ] **Step 5: Verify each preset has exactly one entry per `Highlight` enum value**

Run (from the repository root):

```bash
for name in deuteranopia_ tritanopia_ high_contrast_; do
  echo "== $name =="
  awk "/impl::$name = \{/,/^\};/" models/DecodeHighlightingModel.cpp \
    | grep -o 'Highlight::[A-Za-z]*' | sort -u | wc -l
done
```

Expected: `16` printed three times (once per preset). If any count is not 16, find the
duplicated or missing `Highlight::` value by comparing against the enum list in the
Global Constraints section above and fix the array before continuing.

- [ ] **Step 6: Build to confirm the new code compiles**

Run:

```bash
cmake --build build --target wsjt_cxx
```

Expected: build succeeds with no errors. (If `build/` does not exist yet, configure it
first per `INSTALL`.)

- [ ] **Step 7: Commit**

```bash
git add models/DecodeHighlightingModel.hpp models/DecodeHighlightingModel.cpp
git commit -m "feat(colors): add colorblind-accessible decode highlighting presets

Adds three new predefined color presets for decode-text highlighting:
red-green safe (deuteranopia/protanopia), blue-yellow safe
(tritanopia), and high contrast, alongside the existing Default 1/2
presets."
```

---

### Task 2: Add "Apply Accessibility Palette" button and wiring in the Configuration dialog

**Files:**
- Modify: `Configuration.ui:2787-2817` (button row)
- Modify: `Configuration.cpp` (includes, constructor after `ui_->setupUi (this);` at line 1814, new private helper method)

**Interfaces:**
- Consumes: `DecodeHighlightingModel::default_items_deuteranopia ()`, `default_items_tritanopia ()`, `default_items_high_contrast ()` (from Task 1); `Configuration::impl::next_decode_highlighing_model_` (existing member, `DecodeHighlightingModel`); `MessageBox::query_message (QWidget*, QString const&, QString const&)` (existing, `widgets/MessageBox.hpp:31`).
- Produces: nothing consumed by later tasks.

- [ ] **Step 1: Add the new button to the Colors tab in Configuration.ui**

In `Configuration.ui`, find:

```xml
            <item>
             <widget class="QPushButton" name="reset_highlighting_to_defaults2_push_button">
              <property name="text">
               <string>Reset Highlighting to Default 2</string>
              </property>
             </widget>
            </item>
            <item>
             <widget class="QPushButton" name="rescan_log_push_button">
```

Replace with:

```xml
            <item>
             <widget class="QPushButton" name="reset_highlighting_to_defaults2_push_button">
              <property name="text">
               <string>Reset Highlighting to Default 2</string>
              </property>
             </widget>
            </item>
            <item>
             <widget class="QPushButton" name="apply_accessibility_palette_push_button">
              <property name="toolTip">
               <string>&lt;html&gt;&lt;head/&gt;&lt;body&gt;&lt;p&gt;Push to choose a colorblind or low-vision friendly preset for all highlight items above.&lt;/p&gt;&lt;/body&gt;&lt;/html&gt;</string>
              </property>
              <property name="text">
               <string>Apply Accessibility Palette</string>
              </property>
             </widget>
            </item>
            <item>
             <widget class="QPushButton" name="rescan_log_push_button">
```

- [ ] **Step 2: Add the `QMenu` include**

In `Configuration.cpp`, find:

```cpp
#include <QAction>
```

Replace with:

```cpp
#include <QAction>
#include <QMenu>
```

- [ ] **Step 3: Add a shared confirm-then-apply helper method declaration**

In `Configuration.cpp`, find:

```cpp
  Q_SLOT void on_reset_highlighting_to_defaults_push_button_clicked (bool);
  Q_SLOT void on_reset_highlighting_to_defaults2_push_button_clicked (bool);
```

Replace with:

```cpp
  Q_SLOT void on_reset_highlighting_to_defaults_push_button_clicked (bool);
  Q_SLOT void on_reset_highlighting_to_defaults2_push_button_clicked (bool);
  void apply_highlighting_preset (QString const& description, DecodeHighlightingModel::HighlightItems const& preset);
```

- [ ] **Step 4: Implement the helper method**

In `Configuration.cpp`, find:

```cpp
void Configuration::impl::on_reset_highlighting_to_defaults_push_button_clicked (bool /*checked*/)
{
  if (MessageBox::Yes == MessageBox::query_message (this
                                                    , tr ("Reset Decode Highlighting")
                                                    , tr ("Reset all decode highlighting and priorities to Default 1 values")))
    {
      next_decode_highlighing_model_.items (DecodeHighlightingModel::default_items ());
    }
}
```

Replace with:

```cpp
void Configuration::impl::on_reset_highlighting_to_defaults_push_button_clicked (bool /*checked*/)
{
  if (MessageBox::Yes == MessageBox::query_message (this
                                                    , tr ("Reset Decode Highlighting")
                                                    , tr ("Reset all decode highlighting and priorities to Default 1 values")))
    {
      next_decode_highlighing_model_.items (DecodeHighlightingModel::default_items ());
    }
}

void Configuration::impl::apply_highlighting_preset (QString const& description, DecodeHighlightingModel::HighlightItems const& preset)
{
  if (MessageBox::Yes == MessageBox::query_message (this
                                                    , tr ("Reset Decode Highlighting")
                                                    , tr ("Reset all decode highlighting and priorities to %1 values").arg (description)))
    {
      next_decode_highlighing_model_.items (preset);
    }
}
```

- [ ] **Step 5: Build the accessibility menu and attach it to the new button**

In `Configuration.cpp`, find:

```cpp
  ui_->setupUi (this);

  {
    // Make sure the default save directory exists
```

Replace with:

```cpp
  ui_->setupUi (this);

  {
    auto accessibility_menu = new QMenu {ui_->apply_accessibility_palette_push_button};

    auto deuteranopia_action = accessibility_menu->addAction (tr ("Red-Green safe (Deuteranopia/Protanopia)"));
    connect (deuteranopia_action, &QAction::triggered, this, [this] {
        apply_highlighting_preset (tr ("Red-Green safe (Deuteranopia/Protanopia)")
                                   , DecodeHighlightingModel::default_items_deuteranopia ());
      });

    auto tritanopia_action = accessibility_menu->addAction (tr ("Blue-Yellow safe (Tritanopia)"));
    connect (tritanopia_action, &QAction::triggered, this, [this] {
        apply_highlighting_preset (tr ("Blue-Yellow safe (Tritanopia)")
                                   , DecodeHighlightingModel::default_items_tritanopia ());
      });

    auto high_contrast_action = accessibility_menu->addAction (tr ("High contrast"));
    connect (high_contrast_action, &QAction::triggered, this, [this] {
        apply_highlighting_preset (tr ("High contrast")
                                   , DecodeHighlightingModel::default_items_high_contrast ());
      });

    ui_->apply_accessibility_palette_push_button->setMenu (accessibility_menu);
  }

  {
    // Make sure the default save directory exists
```

- [ ] **Step 6: Build the full application**

Run:

```bash
cmake --build build --target wsjtx
```

Expected: build succeeds with no errors (uic regenerates `ui_configuration_dialog.h` from
the modified `Configuration.ui` automatically as part of this build).

- [ ] **Step 7: Manually verify the new button in the running application**

Run the built `wsjtx` binary (path depends on your build directory layout, typically
`build/wsjtx` or `build/wsjtx.app/Contents/MacOS/wsjtx` on macOS), then:

1. Open *File → Settings…* and go to the *Colors* tab.
2. Confirm a new **Apply Accessibility Palette** button appears next to *Reset
   Highlighting to Default 2*.
3. Click it and confirm a menu with three items appears: *Red-Green safe
   (Deuteranopia/Protanopia)*, *Blue-Yellow safe (Tritanopia)*, *High contrast*.
4. Click *Red-Green safe (Deuteranopia/Protanopia)*, confirm a "Reset Decode
   Highlighting" dialog appears mentioning that preset's name, click *Yes*, and
   confirm the highlight list above updates to new colors.
5. Click *OK* to close Settings, reopen it, and confirm the colors persisted.
6. Click the accessibility button again, choose *High contrast*, click *Yes*, then
   click *Cancel* on the Settings dialog. Reopen Settings and confirm the colors
   reverted to whatever was last saved in step 5 (i.e., the cancel discarded the
   unsaved change, matching existing "Default 1/2" button behavior).

- [ ] **Step 8: Commit**

```bash
git add Configuration.ui Configuration.cpp
git commit -m "feat(colors): add Apply Accessibility Palette button to Colors tab

Wires the three colorblind-accessible presets into the Configuration
dialog via a new button and dropdown menu, reusing the existing
confirm-then-replace flow used by the Default 1/2 reset buttons."
```

---

### Task 3: Document the new accessibility presets in the user guide

**Files:**
- Modify: `doc/user_guide/en/settings-colors.adoc:15-16`

**Interfaces:**
- Consumes: none.
- Produces: none.

- [ ] **Step 1: Add a bullet describing the new button**

In `doc/user_guide/en/settings-colors.adoc`, find:

```adoc
* Press the *Reset Highlighting* button to reset all of the color
  settings to default values.
```

Replace with:

```adoc
* Press the *Reset Highlighting* button to reset all of the color
  settings to default values.

* Press the *Apply Accessibility Palette* button to choose a
  colorblind or low-vision friendly color preset: *Red-Green safe
  (Deuteranopia/Protanopia)*, *Blue-Yellow safe (Tritanopia)*, or
  *High contrast*. Like *Reset Highlighting*, this replaces all
  current color settings after you confirm the change.
```

- [ ] **Step 2: Confirm the AsciiDoc renders without errors**

Run:

```bash
grep -n "Apply Accessibility Palette" doc/user_guide/en/settings-colors.adoc
```

Expected: two matches (the bullet heading and the description sentence).

- [ ] **Step 3: Commit**

```bash
git add doc/user_guide/en/settings-colors.adoc
git commit -m "docs: document Apply Accessibility Palette button"
```
