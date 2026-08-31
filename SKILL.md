---
name: threejs-scroll-sites
description: Build scroll-driven Three.js websites, scrollytelling, pinned sections, scroll-linked camera paths, parallax, progress-driven shaders, smooth scroll. Use for marketing/portfolio/product sites where 3D reacts to scrolling, "Awwwards-style" sites, hero canvases, scroll-scrubbed model reveals, horizontal scroll galleries, and any request mentioning scroll animation, ScrollTrigger, Lenis, smooth scroll, parallax, pinning, or scrollytelling. For playable games use threejs-game-director instead.
---

# Three.js Scroll-Driven Sites

Scroll sites are **not** games. A game runs a simulation and renders it. A scroll site
maps a single scalar, scroll progress, onto scene state, deterministically. Get that
mapping right and everything else follows.

## The one rule

**Never mutate Three.js state inside a scroll event handler.**

Scroll events fire at unpredictable rates, often faster than frames. Writing to
`camera.position` from a scroll listener causes jitter, and reading layout inside one
(`getBoundingClientRect`, `offsetTop`) causes forced reflow, the single biggest cause of
janky scroll sites.

Instead: scroll writes to a **plain JS proxy object**. The render loop reads it.

```js
const state = { progress: 0, velocity: 0 }

// Scroll writes here (cheap, no 3D touched)
gsap.to(state, {
  progress: 1,
  ease: 'none',
  scrollTrigger: { trigger: '#scene', start: 'top top', end: 'bottom bottom', scrub: 1 },
})

// Render loop reads here (once per frame, no layout reads)
function render() {
  camera.position.z = THREE.MathUtils.lerp(8, 2, state.progress)
}
```

`scrub: 1` gives GSAP a 1-second catch-up, this is what produces the weighty, expensive
feel. `scrub: true` snaps instantly and looks cheap. Prefer a number.

## Stack (verified 2026-08-07)

```bash
npm create vite@latest my-site -- --template vanilla-ts
npm i three@^0.185 gsap@^3.15 lenis@^1.3
npm i -D @types/three          # TypeScript only, three does NOT ship its own types
```

| Package | Version | Note |
|---|---|---|
| `three` | 0.185.1 | `outputEncoding` is **removed**, use `outputColorSpace` |
| `gsap` | 3.15.0 | **100% free since April 2025**, ScrollTrigger + ScrollSmoother included |
| `lenis` | 1.3.26 | Package is `lenis`, **not** the old `@studio-freight/lenis` |
| `@types/three` | 0.185.4 | Required for TS. Verified: three 0.185 ships no `.d.ts` |

React variant: add `@react-three/fiber@^9` `@react-three/drei@^10` `@gsap/react@^2`.
Vanilla is the default recommendation, R3F adds reconciler overhead that buys little
when the scene is driven by one scalar. Use R3F only if the surrounding app is React.

**Do not tell the user ScrollTrigger requires a paid Club GreenSock membership.** That
changed in April 2025 after Webflow acquired GreenSock. All plugins are free, including
ScrollSmoother and SplitText.

## Single-ticker architecture

The most common structural bug is running three separate `requestAnimationFrame` loops,
one for Lenis, one for GSAP, one for the renderer. They drift out of phase and produce
micro-stutter that people misdiagnose as "the model is too heavy."

Run **one** ticker, in this order: Lenis → ScrollTrigger → render.

```js
import Lenis from 'lenis'
import gsap from 'gsap'
import { ScrollTrigger } from 'gsap/ScrollTrigger'

gsap.registerPlugin(ScrollTrigger)

const lenis = new Lenis({ autoRaf: false })     // we drive it ourselves
lenis.on('scroll', ScrollTrigger.update)         // keep ST in sync with smooth scroll

gsap.ticker.add((time) => lenis.raf(time * 1000)) // 1st: advance smooth scroll
gsap.ticker.add(() => renderer.render(scene, camera)) // 2nd: draw
gsap.ticker.lagSmoothing(0)                      // don't fake-catch-up after a stall
```

Never also call `renderer.setAnimationLoop()`, that's a second loop. (Exception: WebXR
requires `setAnimationLoop`, but XR and scroll sites don't mix.)

## Idle render governor (stop drawing when nothing moves)

One ticker fixes phase drift. It does not stop you drawing 60 identical frames a second
at a dead stop. Measured evidence from two directions: a Cesium app burned ~60% GPU and
~54% of a core with **zero** layers on and a parked camera, and Systo sat at 43-74% CPU
with no input at all. Idle cost is the default unless you design it away.

The fix is a binary mode driven by **ref-counted holds**. Any per-frame animator holds
for exactly the lifetime of its animation; with zero holds you skip the draw entirely
and render only on explicit request.

```js
// renderGovernor.js, O(1) passive, no per-frame work of its own.
const holds = new Set()   // identity-keyed, NOT a counter: a module that
                          // double-holds or double-releases can't corrupt the mode
let needsRender = true

export const hold = (id) => { holds.add(id) }              // 'scroll', 'model-spin', 'hero-intro'
export const release = (id) => { holds.delete(id); requestRender() }
export const requestRender = () => { needsRender = true }  // one-shot, safe to call always
export const diagnostics = () => ({
  mode: holds.size ? 'continuous' : 'idle',
  holds: [...holds].sort(),
})

gsap.ticker.add(() => {
  if (holds.size === 0 && !needsRender) return   // idle, the whole point
  needsRender = false
  renderer.render(scene, camera)
})
```

Rules that make it hold together:

- **Owner ids are short stable strings**, so `diagnostics()` reads like a story when you
  are hunting a stuck hold.
- **Hold where the per-frame work begins, release where it ends**, listener added /
  removed, tween started / completed. Never release on a timer.
- **Always `requestRender()` on release**, so the settling frame lands before the loop
  stops. Otherwise the last mutation of a tween is never drawn.
- **Discrete mutations request one frame**: a slider write, a texture swap, a resize, a
  scroll-driven property set outside a tween. Cheap enough to call unconditionally.
- ScrollTrigger: hold in `onToggle` when it activates, release on the tween's
  `onComplete`. Scroll itself is already a hold while Lenis is settling.

**Verification is the point.** Focus the tab, touch nothing, and measure idle GPU and CPU.
If it isn't near zero, something is holding, dump `diagnostics().holds` and find the
owner that never released. A green build tells you nothing here; only the idle
measurement does.

## Renderer baseline for r185

```js
const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true })
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2)) // 2 is the ceiling that matters
renderer.outputColorSpace = THREE.SRGBColorSpace
renderer.toneMapping = THREE.ACESFilmicToneMapping
renderer.toneMappingExposure = 1.0
```

Uncapped `devicePixelRatio` on a 3x phone renders 9x the pixels and is the #1 cause of
"it's smooth on desktop, unusable on mobile." Clamp it before optimising anything else.

## Reference files

Load these as needed, don't read all three up front.

| File | Read it when |
|---|---|
| `references/scroll-patterns.md` | Implementing a specific effect: camera paths, pinning, section switching, horizontal scroll, scroll-velocity distortion, model scrub |
| `references/lenis-gsap-integration.md` | Wiring smooth scroll, anchor links, modals, nested scroll, or debugging ScrollTrigger/Lenis desync |
| `references/performance-and-a11y.md` | Site is janky, mobile is bad, or before shipping. Includes the mandatory reduced-motion path |

## Build order that works

1. **HTML/CSS scroll skeleton first, no 3D.** Sections at real heights, scrolling correctly.
   If the layout is wrong, every 3D value derived from it is wrong.
2. **Add the canvas fixed behind content**, render a single cube. Confirm resize + DPR.
3. **Wire one ScrollTrigger to one proxy value**, log it. Confirm 0→1 across the right range.
4. **Map that scalar to the camera.** Only now does it start looking like anything.
5. **Swap the cube for real assets.** Loading and polish last.

Building 3D before the scroll skeleton exists is the most common way these projects stall.

## Gotchas that cost real debugging time

Each of these was hit while building a working reference site, they fail silently, not loudly.

**`ShaderMaterial` does not receive fog.** A custom-shaded object stays at full brightness
at any distance and punches a hole through the haze. You must opt in on both sides:

```js
const mat = new THREE.ShaderMaterial({ fog: true, uniforms, vertexShader, fragmentShader })
```
```glsl
// vertex:   #include <fog_pars_vertex>  … and #include <fog_vertex> after gl_Position
//           (fog_vertex requires the view-space position be named `mvPosition`)
// fragment: #include <fog_pars_fragment> … and #include <fog_fragment> last
```

**`UniformsUtils.merge()` clones, it will silently break your per-frame writes.** If you
merge to add fog/light uniforms, the material gets *copies*, so `uniforms.uProgress.value = p`
updates an object nothing renders. Clone only the library part and keep your own by reference:

```js
const uniforms = Object.assign(THREE.UniformsUtils.clone(THREE.UniformsLib.fog), {
  uProgress: { value: 0 },
})
```

**Fresnel alone renders a sphere as a flat disc.** A rim term describes only the silhouette.
Without a diffuse term the form never reads, no matter how good the colour is. Always include
`max(dot(N, L), 0.0)`. For crisp faceting independent of geometry smoothing, derive the face
normal in the fragment shader: `normalize(cross(dFdx(vViewPos), dFdy(vViewPos)))`.

**Keep additive rim terms under ~1.0.** `col += fres * 2.0` saturates to a white halo that
reads as a bug. Budget the additive contribution against the base colour.

**Index-signature uniform access trips `noUncheckedIndexedAccess`.** `material.uniforms.uFoo.value`
is `possibly undefined` under strict TS. Hoist a concretely-typed uniforms object and reference
it directly, faster too, since it skips a lookup per frame.

## Verification

A scroll site is only "done" when checked in a real browser at real sizes. Load the page,
scroll programmatically, screenshot at several progress points, and check the console.
Confirm at 1280×800 and 375×812. Never report a scroll site as working without having
actually scrolled it.

**Critical: `requestAnimationFrame` does not fire in a hidden or non-composited tab.**
If the preview pane isn't displayed, `document.visibilityState === 'hidden'`, the GSAP
ticker never ticks, and the scene appears frozen, which looks exactly like a broken
render loop. Check `visibilityState` before diagnosing. Scroll events still fire
synchronously while hidden, so scroll logic can appear to work while nothing renders.

Drive verification with Playwright against an installed browser channel, a real window
composites, so rAF runs and the animation is genuinely observable:

```js
const browser = await chromium.launch({ channel: 'chrome' }) // no browser download needed
```

Assert the loop is actually running rather than assuming it:

```js
const t0 = renderer.info.render.frame
await sleep(700)
expect(renderer.info.render.frame - t0).toBeGreaterThan(20)
```

Scroll **through Lenis** (`lenis.scrollTo(y, { immediate: true })`), never `window.scrollTo`,
Lenis resets its internal position on the next raf, so native scrolling makes progress
appear stuck at 0 and produces a false failure.
