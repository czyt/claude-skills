---
name: omarchy-plugin
description: Use when creating, cloning, debugging, validating, installing, or publishing third-party Omarchy Quattro shell plugins built with QML and Quickshell, including bar widgets, panels, overlays, menus, services, and full bar replacements.
---

# Omarchy Plugin

## Overview

Omarchy Quattro plugins are QML components loaded by the existing long-running `omarchy-shell` process. Build against the official `manifest.json` contract and the installed Omarchy shell APIs; do not treat a plugin as an independent Quickshell application.

**Reference-only sources:** [references/omarchy-plugin-reference.md](references/omarchy-plugin-reference.md) and [references/quickshell-reference.md](references/quickshell-reference.md). The installed Omarchy branch and its `shell/` source win when versions disagree.

## When to Use

- Create or clone a custom `bar-widget`, `panel`, `overlay`, `menu`, `service`, or `bar` plugin.
- Connect QML entry points to Omarchy bar/panel lifecycle, IPC, settings, or shared shell services.
- Validate, reload, enable, disable, install, update, remove, or publish a plugin.
- Diagnose manifest, discovery, `qmllint`, panel lifecycle, or Quickshell import failures.

Do not use this for a standalone Quickshell configuration intentionally launched with `qs`/ `quickshell`; that is a different deployment model and must not be presented as an Omarchy plugin.

## Required Workflow

### 1. Choose the contract

Start from the closest official plugin. The kind determines the manifest key and entry file:

| `kinds` value | `entryPoints` key | Typical role |
|---|---|---|
| `bar-widget` | `barWidget` | Item placed in the active bar |
| `panel` | `panel` | Persistent or summoned floating surface |
| `overlay` | `overlay` | Fullscreen overlay |
| `menu` | `menu` | Summoned menu surface |
| `service` | `service` | Headless singleton |
| `bar` | `bar` | Full bar replacement; only one is active |

For an existing built-in, clone it into user config:

```bash
omarchy plugin clone omarchy.clock --edit
```

Use the exact ID printed by the command. Work under `~/.config/omarchy/plugins/<plugin-id>/`; never edit packaged Omarchy source. Saving files reloads plugin code; force discovery with `omarchy-shell shell rescanPlugins`.

### 2. Define `manifest.json`

The manifest is at the repository/plugin root. Use a namespaced third-party ID, never an `omarchy.*` ID:

```json
{
  "schemaVersion": 1,
  "id": "io.github.yourname.clock",
  "name": "Custom Clock",
  "version": "1.0.0",
  "author": "Your name",
  "license": "MIT",
  "description": "A clock for the Omarchy bar.",
  "kinds": ["bar-widget"],
  "entryPoints": { "barWidget": "BarWidget.qml" },
  "barWidget": {
    "displayName": "Custom Clock",
    "category": "Time",
    "allowMultiple": false,
    "defaultSection": "center"
  }
}
```

`entryPoints` values must be safe relative paths and match case-sensitive filenames. Plugin folders must not contain symlinks. A clone may retain `omarchy.clonedFrom` during development; remove it and choose the permanent namespaced ID before publishing.

### 3. Implement inside the existing shell

For a simple bar widget, use Omarchy's `BarWidget` and `WidgetButton` types rather than creating a new `PanelWindow` or starting another Quickshell process:

```qml
import QtQuick
import Quickshell
import qs.Ui

BarWidget {
  id: root
  moduleName: "io.github.yourname.clock"
  implicitWidth: button.implicitWidth
  implicitHeight: button.implicitHeight

  SystemClock {
    id: clock
    precision: SystemClock.Minutes
  }

  WidgetButton {
    id: button
    anchors.fill: parent
    bar: root.bar
    text: Qt.formatTime(clock.date, "HH:mm")
    tooltipText: "Custom Clock"
  }
}
```

For a bar widget with a popup, keep the popup as a nested `Panel.qml` loaded with `Qt.resolvedUrl("Panel.qml")`. Use the same `moduleName` in both files; forward `opened`, `open()`, `close()`, `toggle()`, `closeForPopoutSwitch()`, and the injected `bar`, `anchorItem`, and `hostWidget`. Use Omarchy's `KeyboardPanel` and `PanelKeyCatcher` so bar anchoring, Escape, focus, and panel handoff remain intact.

Use Quickshell modules deliberately: `QtQuick` for QML items, `Quickshell` for shell primitives such as `SystemClock`, `Quickshell.Wayland` for layer-shell windows, and the modules exposed by the installed Omarchy shell for `qs.*` types. Do not copy a standalone `ShellRoot`/`PanelWindow` architecture into a plugin unless the selected Omarchy kind explicitly requires it.

### 4. Validate before enabling

```bash
PLUGIN_ID="io.github.yourname.clock"
PLUGIN_DIR="$HOME/.config/omarchy/plugins/$PLUGIN_ID"

omarchy plugin validate "$PLUGIN_DIR"
qmllint -I "$OMARCHY_PATH/shell" +  "$PLUGIN_DIR/BarWidget.qml"
```

Both commands must exit successfully. If `OMARCHY_PATH` is unset, locate the installed Omarchy shell source/import path first; do not guess an unrelated Quickshell import directory.

| Failure | Recovery |
|---|---|
| Manifest parse/required-field error | Fix JSON and required `schemaVersion`, identity, `kinds`, and `entryPoints`. |
| Entry point file not found | Match the manifest path and on-disk filename, including capitalization. |
| Valid but not listed | Run `omarchy-shell shell rescanPlugins`, then `omarchy plugin list --json`. |
| Listed but not visible | Enable it, confirm the declared kind, and inspect `qs log -p "$OMARCHY_PATH/shell" --tail 100`. |
| Popup opens only once | Forward the panel lifecycle and use the host bar widget as `owner`. |
| QML type/import error | Check the installed Omarchy branch and Quickshell version; do not invent a replacement type. |

### 5. Exercise the lifecycle

```bash
omarchy plugin list --json
omarchy-shell shell summon "$PLUGIN_ID" '{}'
omarchy-shell shell hide "$PLUGIN_ID"
```

Test the actual interaction surface: click, Escape, shell summon/hide, disable, re-enable, shell restart, rescan, and removal. A bar widget should show its ID, kind, and `enabled: true` in the JSON listing.

### 6. Prepare and publish

Before sharing, replace the clone ID, remove clone-only metadata, and keep the repository root installable:

```text
custom-clock/
├── manifest.json
├── BarWidget.qml
├── Panel.qml        # only if the plugin has a popup
├── README.md
└── LICENSE
```

The publishing gate is a public GitHub repository, valid root `manifest.json`, README, license, safe install/removal behavior, and a validated current commit:

```bash
omarchy plugin add https://github.com/yourname/custom-clock.git --enable --yes
omarchy plugin update io.github.yourname.clock --yes
omarchy plugin remove io.github.yourname.clock
```

Submit a marketplace listing through the [plugin marketplace issue form](https://github.com/HANCORE-linux/omarchy-plugin-marketplace/issues/new?template=submit-plugin.yml) only after the repository is usable without maintainer-only context.

## Red Flags — STOP

- Starting `qs`/`quickshell` as a second process for an official Omarchy plugin.
- Inventing `plugin.json`, `runtime`, `entry`, an installer hook, or a systemd service instead of using `manifest.json`.
- Using an `omarchy.*` ID, copying an official ID, or keeping `omarchy.clonedFrom` in a published plugin.
- Editing packaged Omarchy source instead of cloning into `~/.config/omarchy/plugins/`.
- Adding symlinks or unsafe/non-relative entry paths.
- Enabling code before reviewing it. Plugins run unsandboxed inside `omarchy-shell` with the user's permissions.
- Copying an official example's repository URL, author, description, or identity unchanged.

## Output Contract

When creating a plugin, return: the selected kind and why, the complete file tree, each changed file, exact validation commands, observed test steps, install/enable commands, and known version assumptions. If the request is actually for an independent Quickshell config, state that boundary and use a separate standalone workflow.

