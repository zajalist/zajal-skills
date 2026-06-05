---
name: lens-studio-mcp-editing
description: Use when editing a Lens Studio scene through the MCP server — setting properties, wiring component/asset references, attaching scripts, or navigating the scene graph. Covers the component-by-component rule, the reference "{}" echo, and script attachment.
---

# Lens Studio MCP editing

The grammar of `mcp__lens-studio__*`. Read this before touching `SetLensStudioProperty`.

## ⭐ Rule #1: vectors / colors / transforms are set COMPONENT-BY-COMPONENT as `number`

`SetLensStudioProperty` **silently no-ops** when you pass a vec2/vec3/vec4/color/transform as an
object `{x,y,z}` or an array. It echoes `Property X set to {}` and **nothing changes**. Always set
each scalar component in its own call:

| Goal | propertyPath (valueType `number`) |
|---|---|
| Move | `localTransform.position.x` / `.y` / `.z` |
| Scale | `localTransform.scale.x` / `.y` / `.z` |
| Rotate (euler°) | `localTransform.rotation.x` / `.y` / `.z` |
| Text / fill color (RGBA 0–1) | `textFill.color.x` `.y` `.z` `.w` |
| Outline color | `outlineSettings.fill.color.x` … |
| Material color | `mainMaterial.passInfos.0.baseColor.x` `.y` `.z` |

A **successful** scalar set echoes the value (`Property x set to 4`). Seeing `{}` for a vector = it
didn't take. **Always read back** a vector edit you care about.

Scalars, bools, strings, enums, and references all work in a single call as usual.

## What each valueType means

- `number`, `string`, `boolean` — literal.
- `enum` — pass the integer; some enums also need `enumType` (e.g. blend modes). Many enums (text
  alignment, blend mode) are stored as plain ints and accept `valueType: "number"` with the int value.
- `reference` — value is the **target object/component/asset UUID string** (not an object). This is
  how you point a RenderMeshVisual at a material, an Image at a texture, or a script `@input` at a
  SceneObject. **The echo for a reference is also `set to {}` — that is NORMAL for references.** Don't
  trust the echo; read the property back: a set reference shows `value:{UUID,name}`, an unset one
  shows `value:null`.
- `layer_set_mask` — for a SceneObject/Camera/Light `RenderLayer`; pass a mask int, not a LayerSet.

## UUIDs: short prefixes don't resolve

`Get...ById` / `SetProperty` need the **full** UUID (`b0dafd68-fd4c-423b-acc7-475d22f8e2aa`), not the
8-char prefix the scene graph prints. Keep the full ids. Component ids and SceneObject ids are
different — set `text` on the **Text component** id, set `localTransform.*` on the **SceneObject** id.

## Script component properties

For a `ScriptComponent` built from a TypeScript `@component`, set its inputs with the **input name as
the path directly**: `propertyPath: "screenHome"`, `valueType: "reference"`, value = SceneObject UUID.
Not `inputs.screenHome`, not `screenHome.value`. For struct inputs you may set simple leaves
(`myStruct.a`) but set vec/transform leaves component-by-component.

### Attaching a TS script to an object (the full dance)

1. Write the `.ts` into the project (see lens-studio-assets / preview-verify). LS auto-compiles it.
2. `ListLensStudioAssets nameFilter:"MyScript" includeUUID includeType` → grab the `TypeScriptAsset`
   UUID.
3. Create the host object + component:
   `CreateLensStudioSceneObject "Controller"` → `CreateLensStudioComponent ScriptComponent` on it.
4. `SetLensStudioProperty <componentUUID> scriptAsset <tsAssetUUID> reference`. (Echoes `{}` — fine.)
5. Re-fetch the object: the component is now named after your class and exposes every `@input` (e.g.
   `screenHome`, `debugAutoTour`). `inputNames` lists them in order. Now wire each input.
6. **An empty `tsc` log = it compiled.** A reference that "didn't take" usually means the script
   didn't compile, so the input list never appeared — check compile first.

Put controller scripts on an **always-enabled** object. A ScriptComponent on a SceneObject you later
disable stops receiving `UpdateEvent`/events.

## Navigating the scene graph without drowning

`GetLensStudioSceneGraph` on a real project is **hundreds of KB on one line** and will overflow tool
output. Strategies:

- `GetLensStudioSceneGraph depthLimit:2` for the top-level map (still large; it gets saved to a file
  you can parse).
- Parse the saved JSON with a tiny script (`sceneTree` → recurse `children`; each node has
  `id`, `name`, `enabled`, `components[]`, `localTransform`). This is the fastest way to get full
  UUIDs + a tree with component types.
- `GetLensStudioSceneObjectById <fullUUID>` for one subtree (includes children + every component's
  full property bag — big but precise).
- `GetLensStudioSceneObjectByName` when you don't have the id (pass `maxObjects` if names repeat).

## Duplicate instead of build-from-scratch

`DuplicateLensStudioSceneObject` copies an object **and its children with new UUIDs** — the fastest way
to make a second button/list-item/card that already has the right material, mesh, and layout. Then
`SetLensStudioParent` to move it, re-fetch to get the new child UUIDs, and retarget text/positions.
Note: duplication captures state at call time — edits you batch *after* the duplicate in the same
message land on the original, not the copy.

## Gotchas

- `SetLensStudioParent` preserves the object's **local** transform, so a duplicated child keeps its
  old local pos relative to a new parent — reposition after reparenting.
- `DuplicateLensStudioAsset` wants `assetUUID` (not `assetId`); `MoveLensStudioAsset` wants
  `destinationPath` (not `destinationFolderPath`).
- The MCP **port changes on every Lens Studio restart** (50049 → 50050 → …). Re-point the server when
  calls start failing: `claude mcp remove lens-studio -s local` then re-add with the new port + bearer.
- Related: [lens-studio-preview-verify](../lens-studio-preview-verify/SKILL.md),
  [lens-studio-sik-interactivity](../lens-studio-sik-interactivity/SKILL.md).
