---
name: studio-ui
description: Use when creating, restyling or editing any player-facing UI or world-space text in a Roblox game (HUD, screens, panels, buttons, billboards, prompts, notifications, meters). Covers authoring UI in Studio, the house text style and named TextStyle system, proportional scaling, baked icon outlines and shadows. Layout across devices, safe areas and gamepad focus are in roblox-gui.
---

# Studio UI

`SK` = this skill's folder (the base directory shown when it loads). The project's vault preferences (text style, panel style, no all caps) and its own UI notes take precedence over this skill. Use `roblox-gui` for layout across devices, safe areas and gamepad focus.

## Authoring rules

- Author UI in Studio through the Roblox Studio MCP (`execute_luau` in the Edit datamodel). Code finds elements by name and checks the class with `IsA`; a missing element warns and disables that piece, never errors.
- Screens live in `StarterGui`, each driven by a module in `src/client/Controllers/UIController/Screens/` (see AGENTS.md). World UI templates (labels, prompts) live in `ReplicatedStorage.Assets` and are fetched by name, never by path.
- Studio edits are not in git. Tell the user to save the place file after UI changes.
- Studio must be stopped (Edit mode) before editing UI. If `execute_luau` says the Edit datamodel is unavailable in Play mode, stop play with `start_stop_play` (`is_start: false`) first.
- After changing files on disk, wait a few seconds in Edit mode for Rojo to sync before starting Play; edits made during Play are lost when it stops.
- Keep tunable numbers in `src/shared/Config`.

## House text style

- Default to a coloured stroke and a gradient text fill on a white-based TextLabel: base `TextColor3` white, a vertical `UIGradient` (rotation 90: darker edges, lighter centre), and a `UIStroke` in a dark shade of the same hue.
- Plain white with a black stroke only when the user asks for it (body text in dark panels usually is).
- Colours default to max saturation; gradients keep both stops saturated and vary brightness only.
- Headers get a drop shadow: a black copy named `TitleShadow` behind the title, 40% transparent, offset 2px right and 3px down.
- No all-caps text ("Rebirth", not "REBIRTH").
- Don't put a gradient on a TextButton whose background is visible; the gradient tints the background too. Use a coloured text stroke instead.

## Named text styles

A project that uses more than a couple of text colours should name them instead of hardcoding gradients on labels. Chop a Rock's version, worth porting when a project needs it:

- `src/shared/Config/TextStyleConfig.luau` holds `Styles`, each a gradient (top, middle, bottom) plus a stroke colour, e.g. `Chrome` (light grey to white, black stroke; headers), `Gold` (orange to light yellow, brown stroke), `Green` (money).
- `src/shared/Utils/Typography.luau`: `Typography.Apply(root)` walks a GUI, sets the project font and applies the style named by each label's `TextStyle` string attribute (and the `Italic` boolean attribute). `Typography.ApplyTextStyle(label, name)` does one at runtime. `UIController` calls `Apply` on every screen; world templates call it themselves.
- To use one: give the label a `UIGradient` and a `UIStroke`, then set `TextStyle`. To add one, add an entry to `TextStyleConfig.Styles`.

## Scaling

- One layout on every device: build at pixel (Offset) sizes for a 1920x1080 screen, and give each root frame a `UIScale` named `ScreenScale` set to `min(viewport.X / 1920, viewport.Y / 1080)` (Chop a Rock's `Utils/ViewportScale.Bind(gui, trove)` does this). A phone then shows the PC UI shrunk 1:1. Decide this with the user at the start of a project; it's a design choice, not a given.
- Text widths inside a scaled container: `TextBounds` is in on-screen pixels, so divide by the `ScreenScale` value before using it in an `Offset`.
- A ScreenGui that must cover the whole screen (backdrops, fades) needs `IgnoreGuiInset` on, `ScreenInsets` None and `ClipToDeviceSafeArea` off.
- Touch: every keyboard-only action needs a touch path. ContextActionService bindings with `createTouchButton` true plus `SetTitle` give phones an on-screen button.
- Never park a layout object (`UIListLayout`, `UIGridLayout`) directly in a ScreenGui: it lays out the ScreenGui's children and collapses the container.

## Icons

- Bake the black outline into the PNG instead of stacking offset copies: `py -3 SK/scripts/bake_outline.py <pngs> --out <dir>` (512 px, outline 3.4% of the size, round and anti-aliased). Icons shown at 72 px or smaller read better with a thicker outline (about 7%). `UIStroke` can't outline an image's shape.
- Upload icons as `Image` assets under the group through Open Cloud (see the `vfx` skill's `upload_images.ps1` or the `item-icons` skill), never with `upload_image`, which puts them on the user's personal account where a group game can't load them.
- Fresh uploads sit in moderation ("Reviewing") and show blank until approved; check with `GET https://apis.roblox.com/assets/v1/assets/<id>` (`moderationResult.moderationState`). Check an id loads with `ContentProvider:PreloadAsync` (status `Success`).

## Shadows

- HUD rows: `IconShadow` (black copy of the icon, about 3px right and 4px down) and `<Label>Shadow` (black copy of the label, 2px right and 3px down). Keep the shadow text in sync whenever code sets the label. `ZIndex` order: backdrop 1, shadows 2, labels 3, icon 4.

## Buttons

- Sharp corners are the house default (no `UICorner`) unless the project's panel style says otherwise.
- Give hovered buttons room: lay out button lists with `UIGridLayout` so a hover grow doesn't push neighbours.
- A button's clickable area is an invisible `TextButton` named `Button` covering the frame, so the visuals can change freely.
