# Lenis + GSAP ScrollTrigger Integration

Package is **`lenis`** (v1.3.26). The old `@studio-freight/lenis` name is deprecated —
installing it gets you a stale build.

---

## Canonical wiring

```js
import Lenis from 'lenis'
import 'lenis/dist/lenis.css'      // required for anchor/stop behaviour to look right
import gsap from 'gsap'
import { ScrollTrigger } from 'gsap/ScrollTrigger'

gsap.registerPlugin(ScrollTrigger)

const lenis = new Lenis({
  autoRaf: false,        // we drive raf from the GSAP ticker
  duration: 1.2,
  smoothWheel: true,
  syncTouch: false,      // leave touch native — see "Touch" below
})

lenis.on('scroll', ScrollTrigger.update)

gsap.ticker.add((time) => lenis.raf(time * 1000))  // GSAP ticker is seconds, Lenis wants ms
gsap.ticker.lagSmoothing(0)
```

Three things people get wrong here:

1. **`time * 1000`** — GSAP's ticker passes seconds, `lenis.raf()` expects milliseconds.
   Omitting the conversion makes smooth scroll appear frozen.
2. **`autoRaf: false`** — leaving it default spawns a second rAF loop that fights the ticker.
3. **`lagSmoothing(0)`** — without it, GSAP "catches up" after a frame stall by jumping
   time forward, which desyncs scrub position from actual scroll.

You do **not** need `ScrollTrigger.scrollerProxy()`. That's only for scroll containers
that aren't `window` (e.g. a smooth-scroll library that transforms an inner wrapper).
Lenis scrolls `window` natively, so ScrollTrigger reads it correctly on its own.

---

## Anchor links

Native `href="#id"` jumps bypass Lenis and look broken. Intercept them.

```js
document.querySelectorAll('a[href^="#"]').forEach((a) => {
  a.addEventListener('click', (e) => {
    e.preventDefault()
    lenis.scrollTo(a.getAttribute('href'), { offset: -80, duration: 1.4 })
  })
})
```

`scrollTo` accepts a selector string, element, or number. Useful options: `offset`
(sticky-header compensation), `duration`, `easing`, `immediate: true` (jump, no animation),
`lock: true` (ignore user input mid-scroll).

---

## Modals and scroll locking

```js
function openModal() { lenis.stop();  document.body.dataset.locked = 'true' }
function closeModal() { lenis.start(); delete document.body.dataset.locked }
```

Use `lenis.stop()` — **not** `overflow: hidden` on body. Setting overflow while Lenis is
running causes a scroll-position jump on release.

---

## Nested scrollable areas

Any inner element that should scroll on its own (code block, sidebar, modal body) needs
opting out, or Lenis swallows the wheel event:

```html
<div class="log" data-lenis-prevent>…</div>
```

Variants: `data-lenis-prevent-wheel` and `data-lenis-prevent-touch` for finer control.

---

## Touch devices

Default `syncTouch: false` leaves touch scrolling native. **Keep it that way.** Enabling
`syncTouch` on mobile is the most common cause of "scrolling feels laggy/detached on my
phone" — it replaces the OS's tuned momentum with a JS approximation. Smooth scroll is a
desktop-pointer nicety; mobile already feels good natively.

---

## Refresh after async content

ScrollTrigger measures the document once. Anything that changes height afterwards —
images, fonts, loaded GLTFs, expanding accordions — invalidates every trigger position.

```js
// after your loading manager finishes
ScrollTrigger.refresh()
```

```js
// images without explicit dimensions
window.addEventListener('load', () => ScrollTrigger.refresh())
```

Better: set `width`/`height` (or `aspect-ratio`) on every image so layout is stable before
load and no refresh is needed.

---

## Resize

```js
let t
window.addEventListener('resize', () => {
  clearTimeout(t)
  t = setTimeout(() => {
    camera.aspect = innerWidth / innerHeight
    camera.updateProjectionMatrix()
    renderer.setSize(innerWidth, innerHeight)
    ScrollTrigger.refresh()
  }, 150)
})
```

Debounce it. `ScrollTrigger.refresh()` re-measures every trigger and is expensive —
firing it per resize event during a window drag will lock the tab.

On mobile, the URL bar showing/hiding fires `resize` constantly. Guard against it by
only refreshing when **width** changes:

```js
let lastW = innerWidth
window.addEventListener('resize', () => {
  if (innerWidth === lastW) return   // height-only change = browser chrome, ignore
  lastW = innerWidth
  /* …refresh… */
})
```

---

## Desync symptoms → causes

| Symptom | Cause |
|---|---|
| Scrub lags behind scroll by a fixed amount | Missing `lenis.on('scroll', ScrollTrigger.update)` |
| Smooth scroll appears dead | Missing `time * 1000` in the ticker |
| Micro-stutter, worse under load | Two rAF loops — check `autoRaf` and `setAnimationLoop` |
| Positions correct on load, wrong after images appear | Missing `ScrollTrigger.refresh()` |
| Pin jumps at section boundary | `pinSpacing` mismatch, or CSS `transform` on an ancestor of the trigger |
| Everything breaks on resize | Function-based `start`/`end` without `invalidateOnRefresh: true` |

A `transform`, `filter`, or `will-change` on an ancestor of a pinned element creates a new
containing block and breaks `position: fixed` pinning. If a pin behaves bizarrely, walk up
the DOM looking for one.

---

## Teardown (SPA route changes)

```js
function destroy() {
  ScrollTrigger.getAll().forEach((t) => t.kill())
  gsap.ticker.remove(rafCallback)
  lenis.destroy()
  renderer.dispose()
  scene.traverse((o) => {
    if (o.geometry) o.geometry.dispose()
    if (o.material) [].concat(o.material).forEach((m) => m.dispose())
  })
}
```

Keep a named reference to the ticker callback so it can actually be removed — an inline
arrow function cannot.
