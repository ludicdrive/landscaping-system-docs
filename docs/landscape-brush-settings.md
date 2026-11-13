# Landscape Brush Settings

**Landscape Brush Settings** is a small editor-only plugin for Unreal Engine that remembers the brush, size, strength and falloff of every landscape tool and restores them when you switch tools. It also adds optional unified overrides and reusable presets.

> :bulb: The plugin targets **Unreal Engine >= 5.8**. It is an editor module and adds nothing to packaged games. :bulb:

## The problem it solves

Unreal's landscape tools share a single `ULandscapeEditorObject`. Changing the brush, size, strength or falloff while sculpting therefore carries over to every other landscape tool. Since Unreal does not restore the previous values when you switch back, artists constantly have to re-adjust the brush.

This plugin stores a separate state per tool (`ToolModeName` + `ToolName`) and writes it back to the landscape editor whenever the active tool changes.

## Installation

1. Copy the `LandscapeBrushSettings` folder into `<Project>/Plugins/`.
2. Enable the plugin in **Edit -> Plugins**, or add it to the `Plugins` array of your `.uproject`.
3. Rebuild the editor target.

```text
Plugins/LandscapeBrushSettings/
  LandscapeBrushSettings.uplugin
  Source/LandscapeBrushSettings/
    LandscapeBrushSettings.Build.cs
    Private/
      LandscapeBrushSettings.h/.cpp              # Persistent settings object
      LandscapeBrushSettingsManager.h/.cpp       # Capture / restore / presets
      SLandscapeBrushSettingsPanel.h/.cpp        # Slate panel
      LandscapeBrushSettingsModule.h/.cpp        # Ticker, tab and menu registration
```

## Opening the panel

- The tab opens automatically once when **Landscape Sculpt** mode becomes active.
- Closing the tab is respected for the current sculpt session. Leave Landscape Sculpt mode and enter it again to auto-open it again.
- It can also be opened at any time from **Window -> Landscape Brush Settings**.

The tab is a regular nomad tab and can be docked anywhere in the editor.

## Interface

| Element | Description |
| --- | --- |
| **Brush Set** | Brush type used by the active tool. Some tools offer several brush sets. |
| **Brush** | Brush shape within the selected brush set. |
| **Brush Size** | Brush radius in Unreal units. Stored per tool unless **Use Unified Size** is enabled. |
| **Component Size** | Number of landscape components affected at once by the component brush. Only shown for component brush sets. |
| **Tool Strength** | How strongly the active tool is applied per stroke. Stored per tool unless **Use Unified Strength** is enabled. |
| **Brush Falloff** | Edge softness of the brush (`0` = hard, `1` = fully soft). Stored per tool unless **Use Unified Falloff** is enabled. |
| **Use Unified Brush** | Keep the current brush set and brush when switching landscape tools. |
| **Use Unified Size** | Keep the current brush size when switching landscape tools. |
| **Use Unified Strength** | Keep the current tool strength when switching landscape tools. |
| **Use Unified Falloff** | Keep the current brush falloff when switching landscape tools. |

Every label has a tooltip with a short explanation, so hovering the rows is enough to learn the controls.

## How per-tool restore works

- While Landscape mode is active, the plugin captures the live values of the active tool on every editor tick.
- When the active tool mode or tool changes, the stored state for the new tool is applied **before** the live values are captured again, so the previous tool is not overwritten with the new tool's values.
- The first time a tool is seen it is seeded with the values that are currently live. This keeps the first selection non-destructive.
- States are keyed by tool mode and tool name, so tools with the same name in different modes (for example sculpt and paint variants) keep separate settings.
- All landscape tool modes with brushes are supported, not only sculpt.

### Unified settings

The four **Use Unified ...** toggles decide, per value type, whether a value is shared or restored per tool:

- **Enabled** – the live value is not replaced when the tool changes. In addition, the plugin re-applies the outgoing brush set/brush after the engine's tool switch, because Unreal resets the brush set whenever a tool changes.
- **Disabled** – the stored per-tool value is restored on every tool switch.

The toggles can be changed at any time and are saved immediately.

## Presets

Presets capture the full per-tool state of all known tools plus the four unified toggles.

- **Selecting a preset in the dropdown applies it immediately.** There is no separate Apply button.
- **Save As** stores the current per-tool states and unified options under the name entered in the text field. If the field is empty, a unique name (`Preset 1`, `Preset 2`, ...) is generated.
- **Update** overwrites the selected preset with the current settings.
- **Delete** removes the selected preset.

Applying a preset replaces the current in-memory per-tool states with the preset's states and applies the active tool's state right away. Tools that are not part of the preset fall back to being seeded from the live values the next time they are selected.

## Persistence

Settings are stored per user and per project in:

```text
<Project>/Saved/Config/<Platform>/EditorPerProjectUserSettings.ini
```

On Windows this is usually:

```text
<Project>/Saved/Config/WindowsEditor/EditorPerProjectUserSettings.ini
```

The values live in the section:

```ini
[/Script/LandscapeBrushSettings.LandscapeBrushSettings]
```

It contains the unified toggles, the per-tool `ToolStates`, the named `Presets` and the flat `PresetToolStates` list. Deleting the section resets the plugin to its defaults.

> :bulb: Because the file is a per-user settings file, every artist can keep an individual brush workflow without touching the project's `Default*.ini` files. :bulb:

## Notes and limitations

- The plugin reads and writes `FEdModeLandscape` through the engine's private `LandscapeEdMode.h`. Public data members are used, but an engine upgrade may still require a small adjustment.
- Brush switching is emulated (`CurrentBrushSetIndex`, `CurrentBrush`, `PreviousBrushIndex`, `LeaveBrush()` / `EnterBrush()`) because the landscape setters are not exported from the engine module.
- The landscape editor's own brush type and falloff selectors poll the same values the plugin writes, so they stay in sync.
- The plugin operates on the first level editor instance.
- **Component Size** is only relevant for component brush sets; the row is hidden for all other brush sets.

## Troubleshooting

| Symptom | Check |
| --- | --- |
| Tab does not open | Make sure the plugin is enabled and the editor was rebuilt. Enter Landscape **Sculpt** mode; the tab opens once per sculpt session. |
| Values are not restored | Confirm the tool has brush settings (some tools have no brush UI). Make sure the corresponding **Use Unified ...** toggle is off. |
| Preset does not apply | Select a different preset from the dropdown; selecting the already active entry does not fire a change. |
| Settings lost after restart | Check that the editor can write to `Saved/Config/<Platform>/EditorPerProjectUserSettings.ini`. |
| Brush jumps back after switching tools | This is expected when the matching unified toggle is off; the tool's stored brush is restored. |
