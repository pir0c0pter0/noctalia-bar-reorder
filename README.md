# Noctalia Shell - Bar Widget Drag-and-Drop Reordering

Feature contribution for [Noctalia Shell](https://github.com/nicop2000/noctalia-shell): drag-and-drop reordering of bar widgets.

## Goal

**Merge this feature into the official Noctalia Shell project.**

This enables users to reorder bar widgets within the same section (left, center, right) by long-press dragging. Widgets animate smoothly during drag, a ghost overlay follows the cursor, and the new order persists to `settings.json`.

## How it works

1. Long-press (300ms) on any bar widget to start dragging
2. A ghost snapshot follows the cursor while the original widget dims
3. Neighboring widgets shift with smooth 150ms animations to show the drop position
4. Release to drop — the widget order is saved immediately
5. Press Escape or drag outside the bar to cancel

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

- A bar-level `MouseArea` (`dragTracker`) with `pressAndHoldInterval: 300` detects long-press
- Normal clicks pass through unaffected (the dragTracker has `z: -1`)
- Visual displacement uses `transform: Translate` on `BarWidgetLoader` to work inside `RowLayout`/`ColumnLayout`
- `ShaderEffectSource` captures a ghost snapshot before dimming the original widget
- `ListModel.move()` reorders in-memory, `BarService.moveWidget()` persists to `settings.json`
- Per-screen overrides are deep-copied via `JSON.parse(JSON.stringify(...))` to avoid mutating live settings
- Works for both horizontal and vertical bar orientations
- Taskbar widget is excluded from drag (long-press ignored)
- Sections with a single widget cannot be dragged

## License

This contribution follows the same license as the Noctalia Shell project.
