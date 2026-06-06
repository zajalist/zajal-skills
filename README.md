# zajal-skills

Field-tested **agent skills for building Snap Spectacles / Lens Studio projects through the Lens Studio MCP server** — no clicking, mostly from a coding agent (Claude Code, Codex, etc.).

These were distilled from actually shipping a Spectacles app (a kids' AR "Curiosity Coach") end-to-end via MCP: building a full nested UI, a liquid-glass material, custom ray-marched shaders (an animated ocean, a "matrix" grid), a 3D story stage, SIK hand interactions, and a screen-navigation state machine. Everything here is a thing that bit us and the way out.

## Why this exists

The Lens Studio MCP tools are powerful but quietly sharp-edged. The same call that works for a scalar **silently no-ops for a vector**. Custom shader graphs **can't be authored over MCP at all** — only their code nodes can. The reset-and-read-logs tool **times out** on a healthy project. None of this is documented; all of it is load-bearing. Each skill below is the short version of a long afternoon.

## Skills

| Skill | Use when |
|-------|----------|
| [lens-studio-mcp-editing](skills/lens-studio-mcp-editing/SKILL.md) | Setting any property, wiring references, attaching scripts, navigating the scene graph |
| [lens-studio-custom-shaders](skills/lens-studio-custom-shaders/SKILL.md) | Writing custom-code (GLSL-ish) shader nodes, graph params, animation, masking |
| [lens-studio-sik-interactivity](skills/lens-studio-sik-interactivity/SKILL.md) | Making things tappable with hands (pinch/poke), hover feedback, screen navigation |
| [lens-studio-ui-building](skills/lens-studio-ui-building/SKILL.md) | Building readable spatial UI: glass cards, buttons, text, icons, images |
| [lens-studio-preview-verify](skills/lens-studio-preview-verify/SKILL.md) | Running the preview, reading logs, screenshotting, checking compiles |
| [lens-studio-assets-and-fbx](skills/lens-studio-assets-and-fbx/SKILL.md) | Importing textures/3D models, materials, PBR slots, fur/PBR looks, pulling assets off Drive |
| [lens-studio-networking](skills/lens-studio-networking/SKILL.md) | Calling a backend with `InternetModule.fetch`; the Preview device-only throw; mic ⇄ network exclusivity |

## Environment these were verified against

- **Lens Studio 5.15.4** (`.esproj` projects)
- **Spectacles Interaction Kit (SIK) 0.16.4**, **Spectacles UI Kit 0.1.x**
- Lens Studio MCP server (the `mcp__lens-studio__*` tool family)
- Windows 11; agent = Claude Code

Versions matter: SIK ≤ 0.17.3 / UI Kit ≤ 0.1.6 for LS 5.15.4. Package installs tend to duplicate folders (`SpectaclesUIKit 2.lspkg`), which can break compiles — see [lens-studio-preview-verify](skills/lens-studio-preview-verify/SKILL.md).

## The things that waste the most time

1. **Vectors/colors/transforms set component-by-component as `number`.** Passing `{x,y,z}` echoes `set to {}` and changes nothing. → [editing](skills/lens-studio-mcp-editing/SKILL.md)
2. **You cannot author shader graph nodes over MCP.** You can edit a Custom Code node's `.customCode` text (hot-reloads), but wiring its output into the graph is GUI-only. → [shaders](skills/lens-studio-custom-shaders/SKILL.md)
3. **There is no Save tool.** `Ctrl+S` is GUI-only; MCP scene edits only reach the running lens after a preview **reset**. → [preview-verify](skills/lens-studio-preview-verify/SKILL.md)
4. **Tap = Collider + Interactable, added in `onAwake`.** And the `SpectaclesInteractionKit` scene object must be **enabled** or nothing fires. → [sik](skills/lens-studio-sik-interactivity/SKILL.md)
5. **Text "size" is ~30–90, not centimeters**, and a text background won't draw for whitespace-only text. → [ui-building](skills/lens-studio-ui-building/SKILL.md)
6. **`InternetModule.fetch` is device-only and throws *synchronously* in the Preview** unless Device Type Override = Spectacles; `new Request(url)` isn't a thing (pass a URL string); and a lens can't use the **mic and network together**. → [networking](skills/lens-studio-networking/SKILL.md)
7. **Reusing a custom-code graph material on a second mesh can silently corrupt the shared shader** (effect vanishes everywhere, no log) — build an isolated Image material instead. → [shaders](skills/lens-studio-custom-shaders/SKILL.md)
8. **`CompileWithLogsTool` reports a timeout even when the compile succeeded** — verify from the `tsc` log (`"TypeScript compilation succeeded!"` or an empty result), not the tool's exit. → [preview-verify](skills/lens-studio-preview-verify/SKILL.md)

---

MIT. Contributions welcome — keep entries field-tested (the durable shape of the tool, not task narration).
