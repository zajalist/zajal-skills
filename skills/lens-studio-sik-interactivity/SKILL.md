---
name: lens-studio-sik-interactivity
description: Use when making Lens Studio objects respond to Spectacles hands (pinch/poke/tap) via the Spectacles Interaction Kit — adding Collider + Interactable in code, wiring events, hover/press feedback, and a screen-navigation state machine. Includes the "enable the SIK root" gotcha.
---

# Spectacles hand interactivity (SIK)

How to make any SceneObject tappable with hands and wire it to behaviour. Verified with **SIK 0.16.4**.

## The tap recipe: Collider + Interactable on the SAME object, added in `onAwake`

The canonical world-space tap target = a **Physics.ColliderComponent** (the hit volume) + a SIK
**Interactable** (the event source) on the same object. Add both in code — far more reliable than
trying to author SIK components over MCP:

```ts
import { Interactable } from "SpectaclesInteractionKit.lspkg/Components/Interaction/Interactable/Interactable";

private makeTappable(o: SceneObject, onTap: () => void): void {
  let col = o.getComponent("Physics.ColliderComponent") as any;
  if (!col) {
    col = o.createComponent("Physics.ColliderComponent") as any;
    const shape = Shape.createBoxShape();
    shape.size = new vec3(5.5, 5.5, 3);   // LOCAL units; multiplied by the object's world scale
    col.shape = shape;
    col.debugDrawEnabled = false;
    try { col.fitVisual = false; } catch (e) {}
  }
  let it = o.getComponent(Interactable.getTypeName()) as any;
  if (!it) it = o.createComponent(Interactable.getTypeName()) as any;
  it.onTriggerEnd.add(() => onTap());   // a completed pinch/poke/tap
}
```

- The import path above is the one that resolves; `Interactable.getTypeName()` is what
  `createComponent` / `getComponent` want.
- **Collider box size is in the object's LOCAL space** and is then scaled by the object's world scale.
  For a button whose visible body is a child plane scaled 5 under a root scaled 1.5, a root-local box
  of `(5.5, 5.5, 3)` covers it (≈ 8 world units). Size generously; a slightly big hit target feels
  better than a stingy one.

## ⚠️ Enable the SpectaclesInteractionKit object or NOTHING fires

The `SpectaclesInteractionKit` scene object ships **disabled** in many templates. If it's off, there's
no InteractionManager / interactors, so no Interactable ever receives an event and every tap silently
does nothing. Set it `enabled: true` first. (Its `[REQUIRED] Core` child must be present.)

## Interactable events (all are `.add(callback)`)

- `onHoverEnter` / `onHoverExit` — cursor/hand over the target.
- `onTriggerStart` — pinch/poke began on the target (use for press-in feedback).
- `onTriggerEnd` — **completed tap** → fire your action.
- `onTriggerEndOutside` — press started here, released elsewhere → treat as cancel.

Callbacks receive an `InteractorEvent` you can usually ignore.

## Juicy feedback (cheap, huge perceived-quality win)

Drive scale/lift toward a target each frame instead of snapping — reads as a responsive button:

```ts
// per button: baseScale (vec3), mul=1, target=1, z lift
it.onHoverEnter.add(()  => { st.target = 1.12; st.zTarget = 1.4; });
it.onHoverExit.add(()   => { st.target = 1.0;  st.zTarget = 0;   });
it.onTriggerStart.add(()=> { st.target = 0.93; st.zTarget = -0.6;}); // squash in
it.onTriggerEnd.add(()  => this.navigate(target));
// in UpdateEvent:
const k = Math.min(1, getDeltaTime() * 12);
st.mul += (st.target - st.mul) * k;
o.getTransform().setLocalScale(st.baseScale.uniformScale(st.mul));
```

When an action hides the button (navigation), reset all buttons' `target`/`zTarget` to rest so they
don't reappear frozen mid-pop.

## Screen-navigation state machine (the pattern that ties a UI together)

One controller on an **always-enabled** object owns the screens and shows exactly one:

```ts
@input screenHome: SceneObject; @input screenAsk: SceneObject; /* ... */
@input btnAsk: SceneObject; @input btnBack: SceneObject; /* ... */

onAwake() {
  this.screens = { home: this.screenHome, ask: this.screenAsk, /* ... */ };
  this.wire(this.btnAsk, "ask");          // makeTappable + navigate
  this.wire(this.btnBack, "home");
  this.createEvent("OnStartEvent").bind(() => this.show("home"));
}
show(name) {
  for (const k in this.screens) this.screens[k].enabled = (k === name);
  this.btnBack.enabled = (name !== "home");   // persistent Back lives OUTSIDE the screens
}
```

- Keep the controller and the persistent Back button **outside** any screen that toggles (disabling a
  parent disables its scripts/colliders). A button parented under a screen is auto–"tap-disabled" when
  that screen hides — handy for per-screen buttons, fatal for global ones.
- Expose a `startScreen` string and a `debugAutoTour` bool: forcing a screen / auto-cycling screens is
  how you **visually verify navigation** when you can't inject a hand event in the editor preview
  (see [preview-verify](../lens-studio-preview-verify/SKILL.md)). You generally cannot script a fake
  pinch, so prove the state machine with these instead.

## Gotchas

- Build buttons as **real nested SceneObjects** (body + label + icon). Add only the *behaviour*
  (collider/interactable/anim) in code. Don't generate the whole UI at runtime — runtime-built objects
  exist only in the Preview, not the editor scene, and can't be hand-tuned.
- `Shape`, `vec3`, `getDeltaTime()` are globals; no import needed. Only `Interactable` is imported.
- Wrap component creation in try/catch and `print` — if the SIK package path is wrong you want a log,
  not a silent dead button. But remember runtime logs may be unreadable (preview-verify) — also keep a
  `debugAutoTour` fallback.
