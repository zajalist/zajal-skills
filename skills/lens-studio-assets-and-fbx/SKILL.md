---
name: lens-studio-assets-and-fbx
description: Use when importing or wiring assets in Lens Studio via MCP — textures (PNG), 3D models (FBX/GLB), materials and their texture slots, PBR base/normal slots, duplicating/moving assets, and pulling private files off Google Drive into the project.
---

# Assets, materials & 3D models in Lens Studio

Getting textures, materials, and 3D meshes into a project and wired up.

## Importing a texture: just drop the file in `Assets/`

Copy a PNG into the project's `Assets/` folder on disk — Lens Studio's file watcher **auto-imports**
it as a `FileTexture`, and **overwriting it hot-reloads**. Verify with
`ListLensStudioAssets assetTypeFilter:FileTexture nameFilter:"..."`. Generate gradients / icons /
rounded buttons with PowerShell `System.Drawing` and drop them straight in.

This same "write into `Assets/`" path is how you create a **new TypeScript** file too (it compiles on
import). The MCP `ReadWriteTextFile` writes project files with a path **relative to Assets**
(e.g. `PiPi/Scripts/Foo.ts`); the plain Write tool to the absolute on-disk path also works and the
watcher picks it up.

## Materials and their texture slots (what IS and ISN'T MCP-settable)

- **Image material** (`ImageMaterialPreset`): has `baseTex` and alpha-blends. Set the visual's
  `mainMaterial` (reference) then `mainMaterial.passInfos.0.baseTex` (reference). Tint via the Image
  component `baseColor`. The plain **Unlit** preset has **no** texture slot — don't use it for images.
- **PBR / Image material texture slots ARE settable**: `mainMaterial.passInfos.0.baseTex` and
  `...normalTex` take texture-UUID references over MCP. Good for swapping real PBR maps (e.g. Poly
  Haven wood/grass) onto a mesh.
- A PBR material's **scalar `baseColor` is NOT reliably settable** via MCP (the Image material's
  `baseColor` is). Plan tints accordingly.
- Colors on materials still follow the component-by-component rule
  ([mcp-editing](../lens-studio-mcp-editing/SKILL.md)): `...baseColor.x/.y/.z` as `number`.

## Fluffy / fur / cloth look: UberPBR + a CC0 fur texture (not the hair presets)

To make an ordinary mesh read as fluffy/furry, the reliable route is **PBR + a real fur texture**, not
the built-in hair shaders:

1. Create an `UberPBRMaterialPreset` material; assign it to the mesh's visual.
2. Set `mainMaterial.passInfos.0.baseTex` **and** `...normalTex` to a tiling **CC0 fur texture** —
   Poly Haven (polyhaven.com) is the easy source; e.g. `curly_teddy_natural` gives a `diff` (→ baseTex)
   and a `nor_gl` (→ normalTex) map, downloadable from `https://dl.polyhaven.org/...`. The normal map
   does most of the "fuzz".
3. High **roughness**, **metallic = 0**, and tint via `baseColor` (component-by-component —
   [mcp-editing](../lens-studio-mcp-editing/SKILL.md)).

**Avoid the built-in hair presets** (`HairGlowingOrange`, `HairCurlyBlue`, …) for this: they expect
groom/strand geometry with hair UVs and look wrong stretched across an arbitrary solid mesh. They're
for actual hair cards, not "make this blob fluffy." Prefer the PBR-fur approach on ordinary meshes.

## Asset housekeeping gotchas

- `DuplicateLensStudioAsset` → key is `assetUUID` (not `assetId`).
- `MoveLensStudioAsset` → key is `destinationPath` (not `destinationFolderPath`).
- Some presets people expect **don't exist** (`BoxMeshPreset`, `NunitoFontPreset`,
  `BalsamiqSansFontPreset`). Check `GetPresetRegistryTool` before assuming a preset name; substitute
  (e.g. a cylinder mesh, `PacificoFontPreset`/`IndieFlowerFontPreset`).
- Organize by creating folders and `MoveLensStudioAsset`-ing into them; keep generated textures and
  models in their own subfolders so re-imports don't collide.

## 3D models (FBX / GLB)

Drop the `.fbx`/`.glb` into `Assets/` → LS imports it as mesh(es) + material(s) + an `ObjectPrefab`.
Instantiate with `InstantiateLensStudioPrefab <prefabUUID> parentUUID:<...>`. Then set its
transform component-by-component. Notes:

- Imported model scale is unpredictable; expect to rescale hard after instantiating. (AI-generated
  meshes via `GenerateFast3DAssets` come in ~1 cm longest-axis but get a default ×100 prefab scale —
  i.e. ~100 cm — so always adjust.)
- A 3D model used as a "logo" reads as genuinely 3D and can idle-spin via a tiny `UpdateEvent` rotation
  — a nicer "3D version" of a flat PNG logo.
- FBX may reference external textures; keep them alongside the FBX in the same folder so the import
  resolves them.

## Pulling PRIVATE files off Google Drive into the project

You'll often be handed a Drive folder of assets (models, logos). Two hard truths:

1. **Routing bytes through the agent's context doesn't scale.** The Drive MCP `download_file_content`
   returns **base64 into your context** — even a 200 KB file is ~50k tokens; an 800 KB FBX is
   unusable. Use it only to *enumerate* (`search_files parentId='<folderId>'`, `get_file_metadata`),
   never to fetch binaries.
2. **Unauthenticated direct download fails for private files.**
   `https://drive.google.com/uc?export=download&id=<ID>` returns a **Google sign-in HTML page**, not
   the file, unless you carry the owner/viewer's session.

The working pattern — download to disk with the user's **browser cookies**, bytes never touching
context:

```python
# inside a browser tool connected to the user's logged-in Chrome:
res = cdp("Network.getAllCookies", {})           # CDP returns httpOnly cookies too (the SID ones)
cookies = {c["name"]: c["value"] for c in res["cookies"] if c["domain"].endswith("google.com")}
s = requests.Session(); s.cookies.update(cookies)
r = s.get("https://drive.google.com/uc?export=download&id="+fid, allow_redirects=True)
if "text/html" in r.headers.get("content-type",""):   # virus-scan / confirm interstitial
    import re, html
    m = re.search(r'action="([^"]+)"', r.text)         # resubmit the confirm form
    act = html.unescape(m.group(1)); params = dict(re.findall(r'name="([^"]+)" value="([^"]*)"', r.text))
    r = s.get(act, params=params, allow_redirects=True)
open(dest, "wb").write(r.content)
```

Prereqs & checks:
- The browser must be reachable over CDP **and logged into the account that can see the folder**.
  Verify: hit `http://127.0.0.1:<port>/json/version`; confirm Chrome is up with remote debugging on.
  Plain Chrome running ≠ debugging enabled — it's a per-profile opt-in (chrome://inspect) or a
  `--remote-debugging-port` launch with a non-default `--user-data-dir`. An isolated debug profile
  won't have the user's Google login, so it can't see private files — you need the **real** profile.
- After download, sanity-check magic bytes (FBX binary starts with `Kaydara FBX Binary`; PNG with
  `\x89PNG`). An HTML blob where a model should be = you got the sign-in page; auth isn't wired.
- If you can't get an authenticated browser, **stop and ask the user** to enable remote debugging /
  start their automation browser, or to drop the files in `Assets/` themselves. Don't burn cycles on
  unauthenticated downloads.

Related: [ui-building](../lens-studio-ui-building/SKILL.md) (using textures as image planes),
[custom-shaders](../lens-studio-custom-shaders/SKILL.md) (PBR/graph materials).
