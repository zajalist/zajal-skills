---
name: lens-studio-networking
description: Use when a Lens / Spectacles lens talks to an external backend via InternetModule.fetch — the Preview is device-only and throws synchronously, `Request` isn't a runtime constructor, and microphone + network APIs are mutually exclusive. Includes the companion-app pattern for voice without an on-device mic.
---

# Networking from a Lens (InternetModule.fetch)

Calling an HTTP backend from a Spectacles lens. The API is small but every step has a non-obvious
trap, and two whole capabilities (mic + network) can't coexist.

## `fetch` is DEVICE-ONLY — the desktop Preview throws unless you force Spectacles

`InternetModule.fetch` does **not** run in the desktop Preview by default. It throws:

> `InternalError: Cannot invoke 'fetch': API not available on the simulated platform`

The fix is in the **Preview panel**: set **Device Type Override → Spectacles**. With any other device
type (or none) the simulated platform has no networking and every fetch dies. Tell the user to set
this; you can't toggle it over MCP.

## The fetch call throws SYNCHRONOUSLY — wrap it in try/catch, not `.catch`

In the Preview the error above is thrown **synchronously from the `fetch(...)` call itself**, before
any promise exists. A `.catch()` on the returned promise will **not** catch it. Always guard the call:

```ts
try {
  const resp = await this.internetModule.fetch(url, { method: "GET" });
  const data = await resp.json();
} catch (e) {
  print("fetch failed: " + e);   // catches the synchronous "not available" throw too
}
```

## `Request` is NOT a runtime constructor — pass a URL string

`Request` appears as a **type** in the `.d.ts`, so it's tempting to write
`internetModule.fetch(new Request(url))`. At runtime that throws:

> `ReferenceError: 'Request' is not defined`

There is no `Request` constructor in LS scripting. **Pass the URL as a plain string** and put method /
headers / body in the options object:

```ts
internetModule.fetch(url, { method: "GET" });
internetModule.fetch(url, { method: "POST", body: JSON.stringify(payload) });
```

Get the `InternetModule` via an `@input internetModule: InternetModule` (wire the asset) or
`require("InternetModule")`.

## ⚠️ Microphone and network are MUTUALLY EXCLUSIVE

A lens that opens the **microphone** (sensitive user data) **cannot also use `InternetModule`** to
reach an external backend. Combining them fails with:

> `Sensitive user data not available in lenses with network APIs`

This is a platform privacy rule, not a bug. The mic is "sensitive data"; an arbitrary external network
endpoint could exfiltrate it, so Snap forbids both in the same lens. **Only Remote Service Gateway /
Snap Cloud are exempt** — those are the sanctioned way to combine mic + cloud.

### The companion-app pattern (voice without an on-device mic)

When you can't use Remote Service Gateway and still want voice, **move the mic off the lens**:

1. A **companion app** (phone or laptop) is the microphone. It captures speech, does STT, and **POSTs
   the text** to your backend.
2. The **lens has no mic** — so it's free to use `InternetModule`. It **polls** the backend for the
   latest text/answer and renders it.

The lens never touches sensitive data, so the network restriction doesn't apply. This is the practical
way to ship a "talk to it" Spectacles experience against your own backend without RSG.

## Checklist

1. `@input internetModule: InternetModule` (or `require`).
2. Preview panel → **Device Type Override = Spectacles** (or fetch is unavailable).
3. `fetch(urlString, { method, body })` — never `new Request(...)`.
4. Wrap the call in **try/catch** (it throws synchronously in Preview).
5. If you need the mic AND a backend: use Remote Service Gateway, or split into a companion app
   (mic + POST) and a poll-only lens.
6. Verify visually — runtime `print` logs are often unreadable
   ([preview-verify](../lens-studio-preview-verify/SKILL.md)).

Related: [preview-verify](../lens-studio-preview-verify/SKILL.md) (Device Type Override lives in the
same Preview panel; reading logs), [mcp-editing](../lens-studio-mcp-editing/SKILL.md) (wiring the
`InternetModule` `@input`).
