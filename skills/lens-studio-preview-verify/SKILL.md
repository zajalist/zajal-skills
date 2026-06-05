---
name: lens-studio-preview-verify
description: Use when running, previewing, and verifying a Lens Studio project edited over MCP — applying scene edits to the running lens, screenshotting the preview, checking that TypeScript compiled, and working around log/compile timeouts. Includes the "no Save tool" reality.
---

# Previewing & verifying Lens Studio work

You're editing blind unless you reset, screenshot, and check compiles. Here's the loop.

## There is no Save tool — and edits need a reset to take effect

- `Ctrl+S` is **GUI-only**; no MCP tool saves the project. Tell the user to press it to persist scene
  edits. A project **reload wipes unsaved scene edits**, so save before anything that reloads.
- MCP scene edits (new objects, property changes, new scripts) only reach the **running preview**
  after a **reset**. The reset tool keeps editor objects and re-runs scripts; a full reload is the one
  that wipes unsaved work.

## Reset the preview with RunAndCollectLogsTool — with TINY params

`RunAndCollectLogsTool` resets the preview (re-runs `onAwake`/`onStart`, applies your edits) and then
collects logs. On a real project the **log collection hangs and the tool times out** — but the reset
still happened. Avoid the timeout by asking for almost nothing:

```
RunAndCollectLogsTool maxRuntimeMs:800 maxLines:3
```

A good result contains `"msg": "Lens has been reset"`. That's your signal the edits are live. If it
still times out, the reset usually occurred anyway — proceed to screenshot.

## Runtime logs are often unreadable — don't depend on `print`

`GetLensStudioLogsTool` for the `preview` category frequently **times out** too, because a single
broken import (e.g. a duplicated `SpectaclesUIKit 2.lspkg`) floods the log. Consequences:

- Don't rely on reading your own `print()` output to confirm runtime behaviour. Verify **visually**
  (screenshot) and **structurally** (read the property/scene back).
- For navigation/state logic you can't see in a still frame, add a `debugAutoTour`/`startScreen` knob
  to the controller and screenshot each state (see
  [sik-interactivity](../lens-studio-sik-interactivity/SKILL.md)).

## Checking that TypeScript compiled

- `CompileWithLogsTool` also **times out** when a bad import floods logs — don't lean on it.
- Instead read just the `tsc` category, tiny window:
  `GetLensStudioLogsTool categories:["tsc"] maxLines:25 timeWindowMs:15000`.
  **An empty `tsc` result = no TypeScript errors** (it compiled clean). Errors show up here as text.
- If a new script's `@input`s never appear on its component, it didn't compile — check `tsc`.

## Screenshotting the preview

The Lens Studio window won't come to the foreground; capture it occluded. A helper script grabs the
window to a PNG:

```
pwsh -File <skills>/lens-studio-mcp/scripts/capture-preview.ps1 -Out D:\tmp\shot.png
```

Then crop/zoom the region you care about (the **Preview** panel, usually the right third) with a tiny
`System.Drawing` script and `Read` the cropped PNG. Tips:

- Capture → **read the pixels** → decide next edit. Re-screenshot after every meaningful change; never
  assume an edit looks right.
- The preview camera **reframes** when scene content changes (disabling a big object, the stage
  auto-fitting). Re-locate your UI each time rather than trusting fixed crop coords.
- If another subsystem (e.g. a 3D stage) renders over your UI, temporarily disable it for a clean UI
  shot, then re-enable it. Restore what you toggled.

## The verification loop, in order

1. Make edits (component-by-component for vectors).
2. **Read back** anything whose echo was `{}` (vectors, references) to confirm it took.
3. `RunAndCollectLogsTool maxRuntimeMs:800` → expect "Lens has been reset".
4. `GetLensStudioLogsTool categories:["tsc"]` → expect empty (compiled).
5. capture-preview → crop → `Read` → judge → iterate.
6. When it looks right, remind the user to **Ctrl+S**.

## Environment notes

- LS 5.x projects are `.esproj`. For LS 5.15.4 use SIK ≤ 0.17.3 / UI Kit ≤ 0.1.6.
- Package installs frequently **duplicate folders** (`... 2.lspkg`); a duplicate with a broken import
  is the usual cause of the log/compile floods above. Removing the stray duplicate restores normal
  logs.
- `GenerateFast3DAssets` / `GenerateTexture` / `GenerateFaceMaskTexture` **often time out** — don't
  put them on the critical path.
- The MCP port changes each LS restart; re-point the server when every call suddenly fails.
