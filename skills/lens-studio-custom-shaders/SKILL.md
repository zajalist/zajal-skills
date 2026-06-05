---
name: lens-studio-custom-shaders
description: Use when writing custom shaders in Lens Studio via MCP — Custom Code (GLSL-like) graph nodes, port declarations, surface/system inputs, animating without a Time node, masking faces, and script-settable graph parameters. Includes why graph wiring is GUI-only.
---

# Lens Studio custom shaders (Custom Code nodes)

How to get real procedural/ray-marched shading (oceans, plasma, "matrix" grids, liquid glass) into a
material from an agent. Verified porting Shadertoy-style GLSL into Lens Studio graph materials.

## The hard boundary: code yes, wiring no

You **can** create/edit a **Custom Code** node's source: it lives in a `.customCode` text file you
edit like any file, and **it hot-reloads** into the running preview.

You **cannot** author the material graph over MCP — adding nodes, and **connecting a Custom Code
node's output into `Shader3D → Color (Pixel)`** (or the vertex `Position Offset`) is **GUI-only**. So:
the human wires the node once; thereafter you iterate purely by editing the `.customCode` text. Plan
for one manual wiring step per material, then full agent control.

## Anatomy of a Custom Code node

```glsl
// FRAGMENT stage. Wire `result` -> Shader3D Color (Pixel).
output_vec4 result;          // an OUTPUT port named "result"
input_float amount;          // an INPUT port named "amount" (becomes a graph pin / param)

void main() {
    vec2 uv = system.getSurfaceUVCoord0();
    result = vec4(uv, 0.0, 1.0);
}
```

- Declare ports with `output_<type>` / `input_<type>` (`vec4`, `vec3`, `float`, …). These become the
  node's pins.
- **Inline comments on a declaration line create GHOST PORTS.** `output_vec4 result; // color` can
  register a phantom port named `result ;` and your real output silently won't connect / params drop
  out of the compiled uniforms. Keep declaration lines comment-free; put comments on their own line.
- Plain GLSL globals like `uniform float x;` or a bare `vec4 result;` are **not** ports — they become
  ghost ports too. Use the `input_`/`output_` forms.

## Reading surface + system inputs (no extra nodes needed)

The `system` object exposes built-ins so a fragment node can be self-contained:

- `system.getSurfaceUVCoord0()` → `vec2` UVs
- `system.getSurfaceNormalWorldSpace()` → `vec3` world normal (normalize it)
- `system.getSurfacePositionWorldSpace()` → `vec3` world position
- `system.getTimeElapsed()` → `float` seconds (see animation caveat)

This lets you do effects (masking, tri-planar, fresnel) without wiring Surface/Normal/UV nodes in.

## Masking faces (e.g. a disc: sea on top, "underwater" on the sides)

Use the world normal to split a mesh's faces in one shader, so a thin cylinder rim stops smearing the
top texture:

```glsl
vec3 N = normalize(system.getSurfaceNormalWorldSpace());
float topMask = smoothstep(0.80, 0.92, N.y);   // ~1 on near-flat top, 0 on vertical sides
vec3 col = mix(sideColor, topColor, topMask);
```

The threshold band (0.80–0.92) is the knob for how aggressively the rim is treated. Make the side
color self-consistent (no bright/white highlights) or seams reappear at the band edge.

## Animation: `getTimeElapsed` works in code, but you often can't drive "time" from outside

Inside a Custom Code fragment shader, `system.getTimeElapsed()` animates fine. But a graph material's
**`time` parameter cannot be driven via MCP/files** from a script in the cases we hit — if you need
script-controlled motion, **animate a transform instead** (e.g. drift a noise-textured plane with a
tiny `UpdateEvent` script) rather than fighting the material clock.

For real geometric displacement (waves that actually stick up), use a **vertex** Custom Code node
writing a **World Position Offset**; have it self-read position via
`system.getSurfacePositionWorldSpace()` and output the full offset position. (Same ghost-port rules.)

## Script-settable graph parameters (the part everyone gets wrong)

To change a shader value from TypeScript at runtime:

1. The parameter must be an `input_<type>` on a Custom Code node **that is wired into the live output
   chain**. An input that dangles (not connected toward the Pixel/Position output) is dropped from the
   compiled uniforms and is unsettable.
2. The settable key is the node input's **ScriptName**.
3. When correct, the param appears in the compiled `.mat` under `Properties:` (NOT `CachedProperties:`).
   If you only see it under `CachedProperties:`, it isn't a live uniform — re-check the wiring and that
   you used `input_float` (not `uniform float`, which makes a ghost port and drops the param).

A working "controls" pattern: a small `@component` (e.g. `GlassControls`) with an `@input material`
and an `@input` settings struct of sliders; on update it pushes each slider into
`material.mainPass.<paramName>`. The struct's slider widgets give designers live knobs.

## Materials a Custom Code shader plugs into

- **Graph / Shader Graph material** — the host for Custom Code nodes. Author code via `.customCode`,
  wire the GUI once.
- Assigning the material to a mesh and swapping the `.customCode` content is how you reuse one wired
  material for different effects (we overwrote one ocean node's file to fix rim masking — no
  re-wiring).

## Workflow checklist

1. Create/locate the graph material; add a Custom Code node in the GUI; wire its `output_` → Pixel
   Color (or Position Offset). Do this once, by hand.
2. Edit the node's `.customCode` file (ports comment-free; use `system.*` inputs).
3. Reset the preview to hot-reload (see [preview-verify](../lens-studio-preview-verify/SKILL.md)).
4. Screenshot to verify — shaders are visual; logs won't tell you it's wrong.
5. Iterate on the file only. Never trust that an inline-commented declaration compiled to a real port.

Related: [lens-studio-ui-building](../lens-studio-ui-building/SKILL.md) (the liquid-glass card uses
this), [lens-studio-assets-and-fbx](../lens-studio-assets-and-fbx/SKILL.md) (PBR texture slots).
