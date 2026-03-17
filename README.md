# Noctalia Shell - Bar Widget Drag-and-Drop Reordering

Feature contribution for [Noctalia Shell](https://github.com/noctalia-dev/noctalia-shell): drag-and-drop reordering of bar widgets.

## Status: WIP — Input Event Blocker

The visual drag infrastructure (ghost overlay, animated offsets, model reordering, settings persistence) is complete and working. However, **drag activation via mouse input is blocked by a fundamental Quickshell/Wayland limitation** — see [Known Limitations](#known-limitations) below.

## What's included

### New file

- `files/Modules/Bar/Extras/BarDragOverlay.qml` — Ghost overlay component using `ShaderEffectSource`

### Patches (for existing files)

| Patch | Target file | Description |
|-------|-------------|-------------|
| `01-BarWidgetLoader.qml.patch` | `Modules/Bar/Extras/BarWidgetLoader.qml` | Adds `visualOffset` property, `Behavior` animation, and `transform: Translate` |
| `02-Bar.qml.patch` | `Modules/Bar/Bar.qml` | Drag state, coordination functions, overlay, cursor tracking MouseArea, Escape handler, vertical section `objectName`s |
| `03-BarService.qml.patch` | `Services/UI/BarService.qml` | `moveWidget()` function for reorder + persist to settings |

## How to apply

1. Copy `BarDragOverlay.qml` to `Modules/Bar/Extras/` in your noctalia-shell directory:
```bash
cp files/Modules/Bar/Extras/BarDragOverlay.qml ~/.config/quickshell/noctalia-shell/Modules/Bar/Extras/
```

2. Apply patches:
```bash
cd ~/.config/quickshell/noctalia-shell
for patch in /path/to/noctalia-bar-reorder/patches/*.patch; do
  patch -p1 < "$patch"
done
```

3. Restart Quickshell:
```bash
quickshell -c noctalia-shell
```

## Technical details

- Visual displacement uses `transform: Translate` on `BarWidgetLoader` to work inside `RowLayout`/`ColumnLayout`
- `ShaderEffectSource` captures a ghost snapshot before dimming the original widget
- `ListModel.move()` reorders in-memory, `BarService.moveWidget()` persists to `settings.json`
- Per-screen overrides are deep-copied via `JSON.parse(JSON.stringify(...))` to avoid mutating live settings
- Works for both horizontal and vertical bar orientations
- Taskbar widget is excluded from drag (long-press ignored)
- Sections with a single widget cannot be dragged

## Known Limitations

### Quickshell PanelWindow Input Event Delivery

Noctalia Shell uses Quickshell's `PanelWindow` with Wayland layer shell (`WlrLayershell`). The bar content window (`BarContentWindow.qml`) is transparent (`color: "transparent"`). This creates a fundamental constraint:

**Mouse press events are ONLY delivered to the deepest MouseArea in the visual tree** — the one defined inside each widget's own QML file. No overlay, wrapper, parent-level, or sibling MouseArea at any z-level receives press events.

#### What was tested (all failed)

| Approach | Result |
|----------|--------|
| Bar-level `MouseArea` with `z: -1` (original patch) | Never receives press — widgets consume it first |
| Bar-level `MouseArea` with `z: 999` | Never receives press |
| `MouseArea` at `z: 9999` on Bar.qml root Item | Never receives press |
| `MouseArea` at `z: 99999` directly in PanelWindow | Never receives press (even with `hoverEnabled: true`) |
| `Rectangle` + `MouseArea` at `z: 99999` in PanelWindow | Never receives press (even with visible background) |
| `TapHandler` with `gesturePolicy: DragThreshold` (passive grab) | Never fires `onPressedChanged` |
| `PointHandler` with passive grab | Never fires `onActiveChanged` |
| `TapHandler` at same level as working `HoverHandler` | Never fires |
| Wrapper component inside Loader (sibling `MouseArea` z:1000 + inner widget Loader) | Never receives press |
| `Qt.createQmlObject` MouseArea injected into loaded widget item | Never receives press |
| `signal.connect()` on widget's own MouseArea (`pressed`, `clicked`, `pressAndHold`) | Silently fails — handlers never fire |
| Explicit `mask: Region { x:0; y:0; width: ...; height: ... }` on PanelWindow | Does not change behavior |
| `BarPill` MouseArea `onPressAndHold` (BarPillHorizontal/Vertical) | Never fires — widget's deeper MouseArea steals the press |

**Only `HoverHandler` works** at parent levels (it tracks hover, not press). Widget-level `onClicked` handlers work because they are defined in the widget's own QML file.

#### Why this happens

Quickshell's `PanelWindow` on Wayland delivers pointer press events exclusively to the deepest visual item with a `MouseArea` at the click coordinates. Z-ordering among siblings, parent-level handlers, and Qt's `propagateComposedEvents` mechanism are all bypassed. This is likely related to how Quickshell interfaces with the Wayland compositor for layer shell surfaces.

### Possible Future Solutions

1. **Upstream Quickshell change**: Add an API to intercept pointer events at the window level before they reach the scene graph, or allow `TapHandler`/`PointHandler` passive grabs to function on layer shell surfaces.

2. **Edit mode via separate PanelWindow**: A dedicated `PanelWindow` (`WlrLayer.Overlay`) overlaying the bar during edit mode. This window owns its own Wayland surface and reliably receives all mouse events. Activation would need a mechanism that doesn't require intercepting presses (e.g., IPC command, keyboard shortcut, or context menu entry added to each widget).

3. **Per-widget modification**: Add `pressAndHoldInterval` to each widget's own MouseArea. This works but requires modifying every widget file and won't apply to new plugins automatically.

4. **Noctalia Shell native infrastructure**: Propose a `BarWidgetBase` component or widget API that includes drag support, so all widgets (including plugins) inherit it.

## License

This contribution follows the same license as the Noctalia Shell project.
