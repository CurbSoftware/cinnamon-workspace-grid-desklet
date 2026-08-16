> **Standalone-repo copy of the development notes.** The full toolchain
> (install script, live test drivers, the sibling Workspace Name applet that
> duplicates the shared modules) lives in the
> [cinnamon-monorepo](https://github.com/CurbSoftware/cinnamon-monorepo)
> `dev-tools/` directory.
# Workspace Grid Desklet - Development Guide

## Table of Contents
1. [Overview](#overview)
2. [Features](#features)
3. [Project Structure](#project-structure)
4. [Development Setup](#development-setup)
5. [Architecture](#architecture)
6. [API Reference](#api-reference)
7. [Settings System](#settings-system)
8. [Coding Patterns](#coding-patterns)
9. [Workspace Management](#workspace-management)
10. [Testing & Debugging](#testing--debugging)
11. [Documentation References](#documentation-references)
12. [Example Code References](#example-code-references)

---

## Overview

The Workspace Grid Desklet is a Cinnamon desktop desklet that displays a clickable grid of workspaces with their names and highlights the active workspace. It allows users to quickly navigate between workspaces by clicking on tiles in the grid.

**UUID**: `cinnamon-workspace-grid-desklet@curbsoftware`

**Type**: Cinnamon Desklet

---

## Features

### Core Features
- **Workspace Grid Display**: Shows all workspaces in a configurable grid layout
- **Click-to-Switch**: Left-click any workspace tile to switch to that workspace
- **Active Workspace Highlighting**: Visually highlights the currently active workspace using CSS pseudo-class `outlined`
- **Workspace Names**: Displays workspace names with optional index prefixes
- **Dynamic Updates**: Automatically updates when workspaces are added, removed, or renamed
- **Add Workspace**: A trailing `+` tile appends a workspace (capped at 36)
- **Remove Workspace**: Right-click a tile → *Remove*, with a confirmation prompt for named workspaces
- **Rename Workspace**: Right-click a tile → *Rename…*, opens a modal entry dialog

### Configuration Options
| Setting | Type | Description | Default |
|---------|------|-------------|---------|
| `layout-mode` | combobox | Grid layout mode: `auto` (near-square) or `fixed` (rows x cols) | `auto` |
| `fixed-rows` | spinbutton | Number of rows in fixed layout mode | `2` |
| `fixed-cols` | spinbutton | Number of columns in fixed layout mode | `2` |
| `show-index` | checkbox | Prefix workspace names with index number | `false` |
| `enable-workspace-editing` | checkbox | Allow adding, removing and renaming workspaces | `true` |
| `show-add-tile` | checkbox | Show the trailing `+` tile | `true` |
| `confirm-remove` | checkbox | Confirm before removing a **named** workspace | `true` |
| `scroll-wheel-behavior` | combobox | Scroll wheel action: `off`, `col`, `row` | `off` |
| `tile-spacing` | spinbutton | Spacing between tiles in pixels | `4` |
| `width` | spinbutton | Desklet width in pixels | `600` |
| `height` | spinbutton | Desklet height in pixels | `400` |

---

## Project Structure

```
cinnamon-workspace-grid-desklet@curbsoftware/
├── desklet.js              # Main desklet implementation
├── workspaceActions.js     # Add/remove/rename logic + pure layout maths
├── renameDialog.js         # Modal rename dialog
├── metadata.json           # Desklet metadata (UUID, name, description)
├── settings-schema.json    # Configuration schema for settings panel
├── stylesheet.css          # CSS styles for the desklet
├── README.md               # User-facing documentation
└── DEVELOPMENT.md          # This file
```

### File Descriptions

#### `desklet.js`
The main entry point containing the `MyDesklet` class that:
- Extends `Desklet.Desklet.prototype`
- Initializes settings bindings
- Creates the UI grid layout, including per-tile context menus and the `+` tile
- Handles workspace switching events
- Manages signal connections for workspace changes

#### `workspaceActions.js`
All workspace mutation logic, with no St/Clutter dependency so the pure parts
can be unit tested headlessly. Wraps Cinnamon's own API defensively — every
export returns `false` on failure rather than letting an exception escape into
a signal handler.

- `addWorkspace()`, `removeWorkspaceByIndex(index, {confirm, onRemoved})`, `renameWorkspace(index, name)`
- `canAdd()`, `canRemove()`, `isValidIndex(index)`, `getWorkspaceCount()`, `getActiveWorkspaceIndex()`
- `computeGridDims(cellCount, mode, rows, cols)` and `planCells(wsCount, showAddTile, rows, cols)` — pure
- `connectNameChanges(cb)` / `disconnectNameChanges(id)` — watches `org.cinnamon.desktop.wm.preferences` `workspace-names`
- `setDependencies(deps)` / `resetDependencies()` — test seam only

#### `renameDialog.js`
`RenameWorkspaceDialog`, a `ModalDialog.ModalDialog` subclass holding an
`St.Entry`. A modal is used rather than an inline entry because desklets live
on the desktop layer, where grabbing and restoring key focus by hand is
unreliable; `pushModal()` takes a stage-wide grab and focuses the entry.
`promptRename(currentName, callback)` is the safe entry point.

> **Both helper modules are duplicated verbatim in the Workspace Name applet.**
> Cinnamon gives xlets no way to import across xlet boundaries, so the copies
> must be kept in sync — the sync check in cinnamon-monorepo's
> `dev-tools/test-workspace-actions.js` asserts they are byte-identical.

#### `metadata.json`
Required metadata file containing:
```json
{
  "uuid": "cinnamon-workspace-grid-desklet@curbsoftware",
  "name": "Workspace Grid",
  "description": "...",
  "icon": "view-grid-symbolic",
  "prevent-decorations": false,
  "multiversion": true
}
```

#### `settings-schema.json`
Defines all configurable settings with types, defaults, and validation.

---

## Development Setup

### Step 1: Installation Directory
Install the desklet to the Cinnamon desklets directory:
```bash
# User installation
cp -r cinnamon-workspace-grid-desklet@curbsoftware ~/.local/share/cinnamon/desklets/

# Or create a symlink for development
ln -s $(pwd)/cinnamon-workspace-grid-desklet@curbsoftware ~/.local/share/cinnamon/desklets/
```

### Step 2: Enable the Desklet
1. Right-click on the desktop
2. Select "Add Desklets"
3. Find "Workspace Grid" in the list
4. Click "Add to Desktop"

### Step 3: Reload After Changes
After modifying code, reload Cinnamon:
```bash
# Method 1: Restart Cinnamon (preserves session)
cinnamon --replace &

# Method 2: Use dbus (if available)
dbus-send --type=method_call --dest=org.Cinnamon /org/Cinnamon org.Cinnamon.Eval string:'global.reexec_self()'
```

### Step 4: View Logs
Debug output is written to the session log:
```bash
# View live logs
tail -f ~/.xsession-errors

# Or use journalctl
journalctl -f | grep -i cinnamon
```

---

## Architecture

### Class Hierarchy
```
Desklet.Desklet (imports.ui.desklet)
    └── MyDesklet
            ├── _init(metadata, deskletId)
            ├── on_desklet_removed()          -> _cleanup()
            ├── _cleanup()                    idempotent teardown
            ├── on_setting_changed()
            ├── _rebuildGrid()
            ├── _createWorkspaceTile(index)
            ├── _createAddTile()
            ├── _onWorkspaceButtonClicked(actor, clickedButton)
            ├── _openTileMenu(button)
            ├── _destroyTileMenu()
            ├── _deferAction(fn)
            ├── _onAddWorkspace()
            ├── _onRemoveWorkspace(index)
            ├── _onRenameWorkspace(index)
            ├── _getWorkspaceCount()
            ├── _getWorkspaceName(index)
            ├── _computeGridDims()
            ├── _update()
            ├── _connectWorkspaceSignals()
            ├── _disconnectWorkspaceSignals()
            └── _onWorkspacesChanged()
```

### Signal Connections
The desklet connects to these signals:
| Signal | Source | Purpose |
|--------|--------|---------|
| `switch-workspace` | `global.window_manager` | Update active highlight |
| `workspace-added` | `global.workspace_manager` | Rebuild grid on workspace add |
| `workspace-removed` | `global.workspace_manager` | Rebuild grid on workspace remove |
| `changed::workspace-names` | `Gio.Settings(org.cinnamon.desktop.wm.preferences)` | Rebuild grid when any workspace is renamed |
| `scroll-event` | `mainContainer` | Handle scroll wheel navigation |
| `destroy` | the desklet itself | Run cleanup immediately instead of 500 ms later |

### UI Components
```
mainContainer (St.BoxLayout, vertical)
    └── table (St.Table, homogeneous)
            ├── buttons[] (St.Button, style_class workspace-button)
            │       └── St.Label (workspace-label)
            └── add tile (St.Button, style_class workspace-add-tile)   <- optional, always last
                    └── St.Label (workspace-add-label)

Main.uiGroup
    └── _tileMenu (PopupMenu.PopupMenu)      <- at most one alive at a time
            ├── Rename...
            ├── Remove
            ├── ---
            └── Add workspace
                    └── label (St.Label)
```

---

## API Reference

### Imports Required
```javascript
const Desklet = imports.ui.desklet;       // Base desklet class
const St = imports.gi.St;                  // Shell Toolkit widgets
const Lang = imports.lang;                 // Language utilities
const Settings = imports.ui.settings;      // Settings management
const Mainloop = imports.mainloop;         // Event loop utilities
```

### Desklet Base Class Methods
From `cinnamon-docs/markdown/cinnamon-js-ui-desklet-Desklet.md`:

| Method | Description |
|--------|-------------|
| `_init(metadata, desklet_id)` | Constructor, call parent with `Desklet.Desklet.prototype._init.call(this, metadata, desklet_id)` |
| `setContent(actor)` | Set the main content actor of the desklet |
| `setHeader(text)` | Set the desklet header text |
| `on_desklet_removed()` | Override to clean up when desklet is removed |
| `on_desklet_added_to_desktop()` | Override to initialize after added to desktop |
| `on_desklet_reloaded()` | Override to handle reload events |
| `destroy()` | Destroy the desklet with fade animation |
| `highlight(boolean)` | Turn on/off visual highlight |

### Desklet Properties
| Property | Type | Description |
|----------|------|-------------|
| `metadata` | dictionary | Desklet metadata from metadata.json |
| `actor` | St.BoxLayout | Main actor of the desklet |
| `content` | St.Bin | Content container |
| `instance_id` | int | Unique instance identifier |

### Settings Class
From `cinnamon-docs/markdown/cinnamon-js-ui-settings-DeskletSettings.md`:

```javascript
// Initialize settings
this.settings = new Settings.DeskletSettings(this, uuid, instanceId);

// Bind a property
this.settings.bind("setting-key", "propertyName", callbackFunction);

// Alternative binding with direction
this.settings.bindProperty(
    Settings.BindingDirection.IN,  // IN, OUT, or BIDIRECTIONAL
    "setting-key",
    "propertyName",
    callbackFunction,
    null  // optional user data
);
```

### Workspace Manager API
```javascript
// Get workspace count
global.workspace_manager.n_workspaces

// Get workspace by index
global.workspace_manager.get_workspace_by_index(index)

// Get active workspace index
global.workspace_manager.get_active_workspace_index()

// Activate a workspace
workspace.activate(global.get_current_time())

// Get workspace name (custom names)
global.settings.get_strv("workspace-name-overrides")
```

---

## Settings System

### settings-schema.json Structure
From `cinnamon-docs/cinnamon-developer-docs/xlet-settings-ref.md`:

```json
{
  "setting-key": {
    "type": "widget-type",
    "default": "default-value",
    "description": "User-facing description",
    "min": 0,            // for numeric types
    "max": 100,          // for numeric types
    "step": 1,           // for numeric types
    "units": "px",       // for spinbutton
    "options": {},       // for combobox/radiogroup
    "dependency": "other-key",  // conditional visibility
    "tooltip": "Help text"
  }
}
```

### Available Widget Types
| Type | Description | Value Type |
|------|-------------|------------|
| `checkbox` | Boolean toggle | `boolean` |
| `entry` | Text input | `string` |
| `spinbutton` | Numeric input with +/- | `number` |
| `scale` | Slider control | `number` |
| `combobox` | Dropdown selection | `string/number` |
| `colorchooser` | Color picker | `string` (rgba) |
| `filechooser` | File/folder picker | `string` |
| `header` | Section header | N/A |
| `separator` | Visual separator | N/A |
| `button` | Action button | N/A |
| `generic` | Hidden storage | `any` |

### Binding Example
```javascript
// In _init()
this.settings = new Settings.DeskletSettings(this, this.metadata["uuid"], deskletId);

// Simple binding - property auto-updates on change
this.settings.bind("layout-mode", "layoutMode", this.on_setting_changed);

// Property is now available as this.layoutMode
```

---

## Coding Patterns

### Pattern 1: Desklet Initialization
```javascript
function MyDesklet(metadata, deskletId) {
    this._init(metadata, deskletId);
}

MyDesklet.prototype = {
    __proto__: Desklet.Desklet.prototype,

    _init: function(metadata, deskletId) {
        // MUST call parent _init first
        Desklet.Desklet.prototype._init.call(this, metadata, deskletId);

        // Initialize settings
        this.settings = new Settings.DeskletSettings(this, this.metadata["uuid"], deskletId);

        // Bind settings
        this.settings.bind("width", "width", this.on_setting_changed);

        // Create UI
        this.mainContainer = new St.BoxLayout({ vertical: true });
        this.setContent(this.mainContainer);

        // Connect signals
        this.switch_id = global.window_manager.connect('switch-workspace',
            Lang.bind(this, this._update));
    }
};
```

### Pattern 2: Cleanup on Removal
```javascript
on_desklet_removed: function() {
    // Disconnect all signals
    if (this.switch_id) {
        global.window_manager.disconnect(this.switch_id);
    }

    // Remove timeouts
    if (this.timeout) {
        Mainloop.source_remove(this.timeout);
    }
}
```

### Pattern 3: Debounced Updates
```javascript
_onWorkspacesChanged: function() {
    // Prevent rapid successive rebuilds
    if (this._rebuildTimeout) {
        Mainloop.source_remove(this._rebuildTimeout);
    }
    this._rebuildTimeout = Mainloop.timeout_add(100, () => {
        this._rebuildGrid();
        this._rebuildTimeout = null;
        return false; // Don't repeat
    });
}
```

### Pattern 4: Dynamic Grid Layout
```javascript
_rebuildGrid: function() {
    // Clear existing children
    this.mainContainer.destroy_all_children();

    // Create table for grid layout
    const table = new St.Table({ homogeneous: true, reactive: true });
    this.mainContainer.add(table, { expand: true, x_fill: true, y_fill: true });

    const dims = this._computeGridDims();

    for (let r = 0; r < dims.rows; r++) {
        for (let c = 0; c < dims.cols; c++) {
            const i = r * dims.cols + c;
            if (i >= this._getWorkspaceCount()) break;

            const button = new St.Button({ style_class: 'workspace-button' });
            button.index = i;
            button.connect('clicked', Lang.bind(this, this._onButtonClicked));

            table.add(button, { row: r, col: c, x_expand: true, y_expand: true });
        }
    }
}
```

### Pattern 5: CSS Pseudo-Class for States
```javascript
_update: function() {
    const active = global.workspace_manager.get_active_workspace_index();

    for (let i = 0; i < this.buttons.length; i++) {
        if (this.buttons[i].index === active) {
            this.buttons[i].add_style_pseudo_class('outlined');
        } else {
            this.buttons[i].remove_style_pseudo_class('outlined');
        }
    }
}
```

---

## Workspace Management

### Use Cinnamon's API, never raw Meta calls

Cinnamon already handles the fiddly parts. `workspaceActions.js` is a thin
defensive wrapper — do not reimplement any of this:

| Cinnamon API (`/usr/share/cinnamon/js/ui/main.js`) | What it already handles |
|---|---|
| `Main.getWorkspaceName(i)` | Falls back to `"Workspace N"` when unnamed |
| `Main.setWorkspaceName(i, name)` | Pads/trims the names array, stores `""` when the name equals the default, writes the gsetting |
| `Main.hasDefaultWorkspaceName(i)` | Whether a remove needs a confirmation prompt |
| `Main._addWorkspace()` | `append_new_workspace(false, time)` |
| `Main._removeWorkspace(ws)` | **Refuses at `n_workspaces == 1`**, splices the name out, then removes |

Prior art worth reading: `expoThumbnail.js` `onTitleKeyPressEvent()` (Enter
commits, Escape cancels) and `remove()` (confirm only when the name is
non-default).

### Where workspace names actually live

`org.cinnamon.desktop.wm.preferences` key `workspace-names`.

**Not** `org.cinnamon` `workspace-name-overrides` — that key's value on Mint 22
is literally `['DEPRECATED']` and never changes. Earlier versions of this
desklet watched it, which is why renames made elsewhere never refreshed the
grid. Connecting to `changed::<nonexistent-key>` does not throw either; the
handler simply never fires, so this class of bug is silent.

```bash
gsettings get org.cinnamon.desktop.wm.preferences workspace-names
gsettings range org.cinnamon.desktop.wm.preferences num-workspaces   # -> i 1 36
```

The 1..36 range is where `MAX_WORKSPACES` comes from.

### Crash-safety rules for this desklet

These are the rules that keep a workspace edit from taking down the session:

1. **Never hold a `Meta.Workspace` across a timeout or dialog.** Resolve it
   from its index at the moment of use, and re-check `canRemove()` inside the
   confirm callback — the world may have changed while the dialog was open.
2. **Tile menus are parented to `Main.uiGroup`.** `destroy_all_children()` on
   the desklet container will not reach them. `_destroyTileMenu()` must run at
   the top of `_rebuildGrid()` and during cleanup, and it closes the menu
   before destroying it so no grab is orphaned.
3. **Menu actions must not run synchronously.** `PopupMenuBase` closes the menu
   in its *own* `activate` handler, which runs after ours. Destroying the menu
   or pushing a modal before that happens fights the menu's grab. Use
   `_deferAction()`.
4. **`_deferAction()` uses `Mainloop.timeout_add(0, …)`, not `idle_add`.** Idle
   callbacks sit below Clutter's redraw priority, so an action fired while a
   menu or dialog was still animating could be starved for hundreds of
   milliseconds. This was a real, observed failure.
5. **`Desklet.destroy()` defers `on_desklet_removed()` by 500 ms** (an opacity
   ease) but emits `destroy` immediately. Hook `destroy` as well, or handlers
   keep firing against a tearing-down desklet for half a second. `_cleanup()`
   is idempotent and serves both paths.
6. **Every source is tracked.** `_rebuildTimeout`, `_idleSources[]`, all four
   signal ids. Cleanup removes each and nulls it.

---

## Testing & Debugging

### Automated tests

```bash
# Headless unit tests - pure logic, no Cinnamon process needed
gjs test-workspace-actions.js

# Live tests against the running desktop (drives Cinnamon over DBus Eval)
python3 /path/to/cinnamon-monorepo/dev-tools/live-test-desklet.py
```

The live driver exercises the full edge-case matrix — add, remove, rename,
confirmation dialogs, the tile menu lifecycle, the 36-workspace ceiling,
overflowing fixed grids, rapid add/remove bursts, and teardown — then asserts
that workspace count, names and the active index are all restored and that
`~/.xsession-errors` gained no new `JS ERROR` lines. It only ever removes
workspaces it appended itself, so no windows are relocated.

`St` cannot be instantiated outside the Cinnamon process (`libst.so` is not on
the loader path), which is why widget behaviour is covered by the live driver
rather than the headless one.

### Debug Logging
```javascript
// Basic logging
global.log("Message");

// Error logging
global.logError("Error message");

// Detailed logging with context
global.log("[workspace-grid] Rebuilding grid with " + count + " workspaces");
```

### View Logs
```bash
# Real-time log monitoring
tail -f ~/.xsession-errors | grep -i workspace

# Journal logs
journalctl -f --user-unit cinnamon
```

### Common Issues

#### Issue: Desklet crashes on load
**Cause**: Syntax error or missing import
**Solution**: Check `~/.xsession-errors` for stack trace

#### Issue: Settings don't update
**Cause**: Binding key mismatch between settings-schema.json and code
**Solution**: Ensure setting key names match exactly

#### Issue: Grid doesn't rebuild on workspace changes
**Cause**: Signal not connected or disconnected prematurely
**Solution**: Verify signal connections in `_init` and cleanup in `on_desklet_removed`

### Launch Settings Panel
```bash
cinnamon-settings desklets cinnamon-workspace-grid-desklet@curbsoftware
```

---

## Documentation References

### Cinnamon Developer Docs (Local)
| Document | Path | Content |
|----------|------|---------|
| Settings Tutorial | `cinnamon-docs/cinnamon-developer-docs/xlet-settings.md` | Settings API overview and binding |
| Settings Reference | `cinnamon-docs/cinnamon-developer-docs/xlet-settings-ref.md` | Widget types and options |
| Importer Guide | `cinnamon-docs/cinnamon-developer-docs/importer.md` | Module import system |

### API Documentation (Local Markdown)
| Document | Path | Content |
|----------|------|---------|
| Desklet Class | `cinnamon-docs/markdown/cinnamon-js-ui-desklet-Desklet.md` | Base desklet API |
| DeskletSettings | `cinnamon-docs/markdown/cinnamon-js-ui-settings-DeskletSettings.md` | Settings provider |
| SignalManager | `cinnamon-docs/markdown/cinnamon-js-misc-signalManager-SignalManager.md` | Signal management utility |
| PopupMenu | `cinnamon-docs/markdown/cinnamon-js-ui-popupMenu-section.md` | Menu widgets |

---

## Example Code References

### Example Desklets (Local)
| Example | Path | Features Demonstrated |
|---------|------|-----------------------|
| Command Result | `cinnamon-docs/cinnamon-developer-docs/example-desklets/commandResult@ZimiZones/` | Basic desklet, command execution |
| Simple System Monitor | `cinnamon-docs/cinnamon-developer-docs/example-desklets/simple-system-monitor@ariel/5.8/` | Settings binding, periodic updates, multiple widgets |
| Show Remote IP | `cinnamon-docs/cinnamon-developer-docs/example-desklets/show-remote-ip-desklet@nejdetckenobi/` | Network requests, settings |
| Mintoo | `cinnamon-docs/cinnamon-developer-docs/example-desklets/mintoo@sujitagarwal/` | Button actions, images |

### Key Patterns from Examples

#### From simple-system-monitor (comprehensive settings)
```javascript
// Multiple settings bindings with callbacks
this.settings.bindProperty(Settings.BindingDirection.IN, "font_color", "font_color", this.setupUI);
this.settings.bindProperty(Settings.BindingDirection.IN, "width", "width", this.setupUI);

// Global setting monitoring
global.settings.connect('changed::desklet-decorations', Lang.bind(this, this.setupUI));
```

#### From commandResult (periodic updates)
```javascript
_updateWidget: function() {
    this._updateValues();
    this.timeout = Mainloop.timeout_add_seconds(1, Lang.bind(this, this._updateWidget));
}
```

---

## Quick Reference Card

### Essential Imports
```javascript
const Desklet = imports.ui.desklet;
const St = imports.gi.St;
const Lang = imports.lang;
const Settings = imports.ui.settings;
const Mainloop = imports.mainloop;
```

### Minimal Desklet Template
```javascript
const Desklet = imports.ui.desklet;
const St = imports.gi.St;
const Settings = imports.ui.settings;

function MyDesklet(metadata, deskletId) {
    this._init(metadata, deskletId);
}

MyDesklet.prototype = {
    __proto__: Desklet.Desklet.prototype,

    _init: function(metadata, deskletId) {
        Desklet.Desklet.prototype._init.call(this, metadata, deskletId);

        this.settings = new Settings.DeskletSettings(this, metadata.uuid, deskletId);

        let container = new St.BoxLayout({ vertical: true });
        let label = new St.Label({ text: "Hello World" });
        container.add(label);

        this.setContent(container);
    },

    on_desklet_removed: function() {
        // Cleanup
    }
};

function main(metadata, deskletId) {
    return new MyDesklet(metadata, deskletId);
}
```

### Command Reference
```bash
# Reload Cinnamon
cinnamon --replace &

# Open settings
cinnamon-settings desklets UUID

# View logs
tail -f ~/.xsession-errors
```
