---
name: item-icons
description: Use when an item in a Roblox game (pet, creature, tool, egg, collectible, any model) needs its card icon (a PNG shown in inventories, indexes and shops instead of a 3D ViewportFrame), when new items are added, when an item's colour or mesh changed so its icon is stale, or when the user asks to render, re-render, upload or fix item icons. Covers the per-project manifest, reading part colours from Studio into specs.json, rendering 512x512 outlined PNGs in headless Blender, the contact sheet, uploading to the Sciyth Studio group and the generated ItemIconConfig.
---

# Item icons

`SK` = this skill's folder (the base directory shown when it loads). Run every script from the project root: they read and write the project's `assets/Icons/`.

Every item gets a 512x512 transparent PNG icon, rendered in Blender from its FBX and its Studio part colours, and uploaded once to the group. Screens show it in an `Icon` ImageLabel and fall back to a ViewportFrame until an item has one, so a new item never breaks the UI.

The look (approved in Chop a Rock, 2026-10-23): a near-front, bright Cycles render, raised whole studs on stud-variant parts, the drawn Inlet texture on `2022 Inlet` parts, and a black outline of 2.2% that follows the silhouette. All of it is in `STYLE` at the top of `SK/scripts/icon_renderer.py`. A new game may want its own look: ask the user before the first batch, and don't change an approved look without them.

## Files

| File | What it is |
| --- | --- |
| `assets/Icons/manifest.json` (project) | Icon key (`<Kind>/<Key>`, e.g. `Pet/Bloky`) → `{ "fbx": "assets/...fbx", "objects": {} }`. `objects` maps a Studio part name to the FBX object name when they differ |
| `assets/Icons/specs.json` (project) | Per icon, the Studio model's parts: colour, material, variant, size, mesh size, position, orientation. Also the stud variants and their tile textures |
| `SK/scripts/icon_renderer.py` | Renders `assets/Icons/<Kind>_<Key>.png` and `contact_sheet.png` |
| `SK/textures/` | Stud tiles: `SmallStud.png` sampled from Studio, `Inlet.png` drawn to match it, `AnimalStud.png` (stored as 0.5 = x1 brightness) |
| `SK/scripts/upload_icons.py` | Uploads new or changed PNGs and writes `src/shared/Config/ItemIconConfig.luau` |
| `assets/Icons/uploaded.json` (project) | Icon key → `{hash, id}`. Unchanged PNGs aren't uploaded again |
| `src/shared/Config/ItemIconConfig.luau` (project) | Generated `{ <Kind> = { <Key> = "rbxassetid://..." } }`. Don't hand-edit it |

The screens read icons through a small util. Chop a Rock's `src/shared/Utils/ItemIcons.luau` (`Get(kind, key)`, `Apply(holder, kind, key, buildFallback)`) is the one to port the first time a project needs it.

## Adding or refreshing icons

1. **Manifest.** Add `"<Kind>/<Key>": { "fbx": "assets/<Category>/<Name>/<Name>.fbx", "objects": {} }`. Several items may share one mesh at different scales, so check the Studio model's MeshId and size, not the folder name.
2. **Specs from Studio** (Edit mode, `execute_luau`). Walk the item's model and, for every BasePart, record `color` (0-255), `material`, `variant` (MaterialVariant), `size`, `meshSize`, `position` and `orientation`. Write them under `icons["<key>"].parts` in `specs.json`, next to `model` and `boundsSize`. Studio is the source of truth for colours, so never type colours in by hand. On a new project, copy the `variants` block from Chop a Rock's `assets/Icons/specs.json` and trim it to the variants this game uses.
3. **Render** (about 30 s per icon on Cycles; the Blender GUI isn't needed):
   ```bash
   "/c/Program Files/Blender Foundation/Blender 5.2/blender.exe" -b --factory-startup --python "SK/scripts/icon_renderer.py" -- <Kind>/<Key>
   ```
   With no keys it renders everything in the manifest, so run big batches in the background. Every run rewrites `contact_sheet.png`; rendering one kind also writes `contact_sheet_<Kind>.png`.
4. **Check.** Open the PNG and the contact sheet. Compare silhouettes and colours against the Studio model, and send the contact sheet to the user. New items in the approved look are a check, not a gate. A new part type (a new material or variant) needs the user's eye first.
5. **Upload.** `py SK/scripts/upload_icons.py`. It reads the key from the `ROBLOX_API_KEY` user env var, uploads changed PNGs as `Image` assets under group 10962733, records them in `uploaded.json` and regenerates `ItemIconConfig.luau`. `--config-only` only regenerates the config. Then run `./scripts/check.ps1`. Images can report `Failure` for a minute after upload, so retry before assuming a bad id.

Never print, echo or write the API key anywhere. The script passes it to curl through a temporary header file.

## Renderer notes

- Roblox (x, y, z) = Blender (-x, z, y) for FBXs made with `rock_builder.export_fbx` (the `model-workflow` kit). Parts are placed from Studio's size, position and orientation (Ry·Rx·Rz), so the FBX's export scale doesn't matter.
- Parts whose colour comes from vertex colours (meshlab models: one white `Body`) carry `"vertexColors": true` in their spec, which multiplies the FBX colour attribute in. Painted parts use `textureFile` (a PNG with studs baked in) instead.
- Raised studs are fitted per flat region at Studio's tile pitch (`StudsPerTile`), whole studs only, with the grid phase picked to fit the most. Studs that would sit under another part (eyes, props) are skipped.
- The camera direction is `STYLE.camera_direction`, in Roblox axes. The view is zoomed and shifted so the silhouette fills the frame minus the outline and margin, so every icon has the same border.
- Neon parts get emission (`neon_strength`). Separate glowing bits get their own outlines.
