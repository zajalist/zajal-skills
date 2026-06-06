---
name: lens-studio-ui-building
description: Use when building spatial UI in Lens Studio via MCP — cards, buttons, text, icons, images. Covers text sizing, the whitespace-background trap, plane orientation, readable labels, textured/rounded image planes, and renderOrder layering.
---

# Building spatial UI in Lens Studio

Making good-looking, readable in-lens UI out of Text / Image / plane primitives. Pairs with
[lens-studio-sik-interactivity](../lens-studio-sik-interactivity/SKILL.md) for making it tappable.

## Text sizing is ~30–90, NOT centimeters

A Text component's `size` is a font-ish unit roughly **30–90** for normal UI, not world cm. A heading
might be 46–72, a body line ~34–55, a button label ~25–30. If text is microscopic or invisible, it's
almost always a size-unit misunderstanding, not a position bug. (If you build text from a script and
it's tiny, multiply your intended sizes by ~14.)

## The whitespace-background trap

A Text `backgroundSettings` panel (rounded colored fill behind the glyphs) **will not draw for
whitespace-only text**. To make an image-only tile or a solid swatch from a Text background, give it a
real glyph (a `"."`) and hide the glyph with `textFill.color.w = 0`. For anything bigger than a small
square, don't use a Text background at all — use a textured plane (below).

## Readable labels over busy/translucent backgrounds

White text alone vanishes over bright glass or the world. The universal fix is **white fill + a dark
outline**:

```
outlineSettings.enabled = true        (boolean)
outlineSettings.size    = 0.5         (number; 0.25 is too thin, ~0.45–0.6 reads well)
outlineSettings.fill.color.{x,y,z}=0  (black; default is already black)
```

Optionally add `dropshadowSettings` for extra depth. This "caption treatment" makes any label legible
on any background and is the single highest-leverage readability move.

## Planes face sideways by default — rotate them

A `PlaneMeshObjectPreset` lies **flat (horizontal)**. To make a card/button face the camera, set
`localTransform.rotation.x = 90`. After that rotation: **width = `scale.x`, height = `scale.z`**,
thickness = `scale.y`. Put backing cards *behind* content with a small negative local Z.

## Glass / colored cards and pill buttons

- A frosted **glass card** is a rotated plane with a graph "liquid glass" material (see
  [custom-shaders](../lens-studio-custom-shaders/SKILL.md)). Buttons sharing that same glass material
  blend into the card — differentiate them with strong label outlines, a slight scale/lift on hover,
  or a distinct (more opaque/tinted) material per button.
- A **pill/rounded button** = Text + `backgroundSettings` (`cornerRadius`, `fill.color.*`, `margins.*`)
  — but only for small labels with visible glyphs. A full pill = corner radius ≈ ½ the height.
- Compose a button as nested objects: `Btn_X` (root, holds transform + later the collider) → `BtnBody`
  (the glass/colored plane) + `BtnLabel` (Text) + `Icon` (Image or empty slot). This keeps each part
  hand-tunable and lets a nav script add interactivity to the root.

## Images, icons, mascots: use an Image component, not a Text background

For anything wide or full-bleed, a Text-background texture **tiles by world size** (you get repeated
bands; `tileCount`/`textureStretch` don't truly stretch). Use a **textured plane** instead:

1. Plane preset; `rotation.x = 90`; `scale.x` = width, `scale.z` = height.
2. Create an **Image material** (`ImageMaterialPreset`) — it has a `baseTex` slot and alpha-blends.
   (The plain Unlit preset has **no** texture slot.)
3. On the visual: `mainMaterial` = image-mat UUID (reference), then
   `mainMaterial.passInfos.0.baseTex` = texture UUID (reference). Optionally set Image `baseColor` to
   tint.
4. Layer with the visual's `renderOrder` — Image-material planes are depthTest-false, so they sort by
   `renderOrder` like Text, **not** by z. Give labels a higher `renderOrder` (e.g. 20) than their
   card so they draw on top.

For an icon/logo, make the SceneObject an **Image** (RenderMeshVisual + image material), not a literal
Text object with a texture — Text-as-image fights you on sizing and the whitespace trap.

## Rounded corners on a plane

Bake them into the **PNG's alpha**: draw your gradient/art clipped to a rounded-rect path on a
transparent 32-bit bitmap (PowerShell `System.Drawing` works great for generating gradients, icons,
and rounded buttons). The Image material shows the transparency. Overwriting the PNG in `Assets/`
hot-reloads it.

## Layout sanity

- Know your card bounds: a card of `scale.x=34, scale.z=48` (rotated) spans ±17 horizontally, ±24
  vertically in its local space. Place children within that.
- Center a heading by setting the Text `horizontalAlignment = 1` (center) **and** the object's
  `position.x = 0`. Left-aligned + off-center x is why a "title" reads like a corner label.
- The editor **Preview** camera reframes as scene content changes (e.g. disabling a big sibling). The
  UI itself is fine; just re-crop screenshots. Verify look in the Preview panel, not the Scene panel.

## Billboard speech bubble over a target

A floating "speech bubble" that labels a mascot/object and stays readable from any angle is a small
world-space Text driven by an `UpdateEvent` script. The durable shape:

- **Billboard to the camera, yaw-locked.** Each frame point the bubble at the camera with
  `quat.lookAt(dirToCamera, up)` — but **zero the pitch/roll** (build the look direction from the
  camera position with `y` flattened) so the text stays upright instead of tilting when you look up or
  down. Pure billboarding (full lookAt) makes labels lie back and get hard to read.
- **Float above the target.** Position it at the target's world position + a fixed up offset; follow
  the target each frame so it tracks if the target moves.
- **Word-wrap + auto-shrink to a bounded panel.** Word-wrap the text, then shrink the font `size`
  ([sizing is ~30–90](#text-sizing-is-3090-not-centimeters)) until the wrapped block fits a fixed
  panel width/height — so long answers don't blow past the bubble.
- **Grow upward from a fixed bottom edge.** Anchor the bubble's bottom just above the target and let it
  expand up as text gets taller, so it **never overlaps the target mesh** (growing from the center
  would push the bubble down onto the mascot's face).

Give the text the [white-fill + dark-outline caption treatment](#readable-labels-over-busytranslucent-backgrounds)
so it reads over the world, and a textured rounded-rect plane behind it for the bubble body.

## UI Kit widgets (when you want them)

Instantiate prefabs (`LabelledButton`, `LabelledToggle`, `FramePrefab`, `ScrollViewListItem`) via
`InstantiateLensStudioPrefab`; discover exact names with `ListLensStudioAssets` (ObjectPrefab). The
visible label is a child **"Label" Text** — set its `text` and bump its `renderOrder` so it isn't
occluded. Changing a UI Kit button `_style` can make the Label vanish (re-set text + renderOrder).
`ContainerFrame` renders nothing until configured — for a quick card just use a colored/glass plane.
