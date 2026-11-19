---
name: model-workflow
description: Use whenever the user asks to get something modelled for a Roblox game (a prop, creature, pet, tree, vehicle, building, character piece, any 3D asset), asks how to make a model, or mentions meshlab or modelling in code. Lays out the six-step pipeline (idea, ChatGPT reference, ChatGPT design prompt, Claude models it in Blender or as meshlab Luau code, feedback loop, import) and says which step they are on. Sets of models also use batch-model.
---

# Model workflow

`SK` = this skill's folder (the base directory shown when it loads). Run scripts from the project root: they read and write the project's `assets/` folder.

Every model goes through the same six steps. When the user brings up a model, start by listing the steps, say which one they are on, and name what they owe next. Don't skip ahead to Blender without a reference image.

1. **Idea.** The user says what it is, where it goes, roughly how big (studs), and the look (low-poly faceted, plain colours, no baked texture unless asked). Ask for what is missing and recommend defaults. Ask whether it needs to animate (rig) or break apart (pre-split pieces).
2. **ChatGPT reference.** The user collects 2 to 4 reference images (existing game assets, screenshots, Pinterest) and gives them to ChatGPT. Tell them to use references only for colour palette, lighting and finish, never for shape.
3. **ChatGPT design prompt.** Claude writes the prompt that makes ChatGPT produce one multi-view reference sheet (front, back, left, right, top, one 3/4 view). Start from `references/reference-sheet-prompt.md`, fill in the SHAPE block for this model, and keep it short. The user runs it and sends the image back.
4. **Claude models it.** Two routes: Blender (below) or meshlab (see "The meshlab route"); pick one with the user before building. Ask which view is the front (front faces Roblox -Z). Blender work goes through the Blender MCP with `SK/scripts/rock_builder.py` (the house Blender kit; despite the name it builds any prop). Every model: one watertight mesh per part, flat bottom if it sits on the ground, pivot at bottom centre, low tri count, plain colour set on the part in Roblox.
5. **Feedback loop.** Claude renders previews from several angles and shows them. The user says what to change; Claude adjusts and re-renders until the user approves. Player-facing looks need approval before they're final. Don't export until approved.
6. **Import.** Claude exports FBX files to `assets/<Category>/<Name>/` (`rock_builder.export_fbx` scales vertices 2x so 1 Blender unit imports as 2 studs; Roblox ignores the FBX unit option), uploads and loads them (below), wires them in through the Studio MCP (names, parent, tags, attributes, registry entries, colour, material, position), and checks the values. The user playtests and saves the place. Items shown on cards also get an icon with the `item-icons` skill.

## Loading a model into Studio

Claude does this; the user doesn't need the 3D Importer.

1. Ask first, since it uploads to the group: `powershell -File SK/scripts/upload_model.ps1 -Path <fbx> -Name "<Name With Spaces>"`. It needs the `ROBLOX_API_KEY` user env var and prints the asset id. If an upload keeps failing with `INTERNAL`, put spaces in the display name.
2. Studio must be in Edit mode. In `execute_luau`, `InsertService:LoadAsset(id)` returns a Model with one MeshPart per Blender object, named after the object, at the right size and orientation (Blender -Y is Roblox -Z).
3. Read an existing model of the same kind first and match its properties (material, stud variant, collision fidelity, anchoring).

## The meshlab route

meshlab (`SK/meshlab/`, MIT) builds stud-style props as Luau code: a model is a function in `models/<Name>.luau` that builds an EditableMesh from boxes, lathes, tubes and lofts. Lune runs it into an `.obj`, and headless Blender renders it with Studio-like lighting. To use it in a project, copy `SK/meshlab` to the project's `tools/meshlab` (once) and keep that project's models there. `models/` here holds three examples (`ForestTree`, `IntroCar`, `Crate`).

- **When it fits:** blocky, chunky props made of boxes and cylinders (trees, vehicles, crates, signs, decor), and families of variants: `lib.seed` makes each seed a different tree. It's cheap to iterate. **When it doesn't:** curved or bubbly shapes, faceted boulders, anything that splits, and creatures (use Blender).
- **References:** a sheet still helps for anything seen up close. Background decor can skip steps 2 and 3 and restyle an existing model instead.
- **Build and look:** from `tools/`, run `lune run meshlab/run <Name> [seed]`, then `"/c/Program Files/Blender Foundation/Blender 5.2/blender.exe" -b -P meshlab/render.py -- <abs .obj> <abs .png> hero 1.3` (views: hero, front, side, rear). Pass absolute paths; with relative ones Blender renders to nowhere. Read `meshlab/BRIEF.md` and two models before writing one.
- **House rules in a model:**
  - Set `Shapes.PITCH = 1.0` so the stud UVs line up with `2022 Stud` / `2022 Inlet`.
  - Wrap round details (lights, wheels, knobs) in a `plain(fn)` helper that points their UVs at (0.03, 0.03); studs on small curves look bad (see `IntroCar.luau`).
  - Use `beginBake({ cell = 2-2.5 })`; smaller cells split faces and add triangles.
  - Keep the footprint of whatever the model replaces, so code that seats or animates against it keeps working.
- **Import:**
  1. `blender -b -P tools/meshlab/tools/export_fbx.py -- <abs out.fbx> <abs a.obj> ...` gives one object per OBJ, named after the file, so copy an OBJ to `Body.obj` to get a part named `Body`.
  2. Upload and load as above. meshlab FBXs load at 1 unit = 1 stud, with no 2x scaling.
  3. Set `Color` white (it multiplies the vertex colours), Plastic and the stud variant.
- **Facing:** `export_fbx.py` imports the OBJ with `forward_axis="Z"` so the front (-Z) lands on Roblox -Z. Check the facing in Studio against the code that drives the model.

## Sets of models

When the models come as a set (a family of pets, tiers of one prop, a zone's decor), step 3 has two parts. First, one lineup prompt shows the whole set side by side (front views on top, 3/4 views below, ordered by tier or rarity) so the pieces read as a family and the user can approve the set at once. Then write one six-view prompt per item that attaches the lineup and names the item by its position in it. Prompts for a set go in `assets/<Category>/<Set>/Prompts.md`. To build the whole set at once, follow the `batch-model` skill.

## Notes

- Shortcuts exist when a new model is a variant of an existing one: duplicate and rotate, resize or recolour in Studio instead of modelling. Offer this before starting steps 2 to 4.
- Vertex colours baked in Blender show in Roblox when the part colour is white; the part colour multiplies them.
- Buildings and structures can be flat: no baked shading or vertex colours, just `rb.planar_stud_uvs(me)` and the part colour. Check the project's vault for its shading rule.
- Updating a model that's already in Studio: if its Model still has the `RBX_ReimportId` attribute and the mesh names match, the user selects it and presses Alt+Shift+R (Reimport). This keeps colours, materials and children. Otherwise load the new upload and run `target:ApplyMesh(imported)` on each existing MeshPart through `execute_luau`, then delete the loaded Model.
- Blender MCP: don't open a `.blend` and export in the same `execute_blender_code` call (the export loses its context). Open the file in one call and export in the next. Shared `.blend` files hold other models, so edit only the target objects and don't change other objects' visibility before saving.
- FBX export skips objects whose collection is hidden in the view layer and still writes a near-empty file with no error. Unhide the collection before `export_fbx`, then check the file by re-importing it and counting the objects.
- `assets/` is the hand-off folder. Commit with explicit paths.
