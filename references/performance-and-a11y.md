# Performance & Accessibility

Scroll sites fail on mid-range Android far more often than on the developer's machine.
Work this list in order — the first three items account for most real-world jank.

---

## 1. Clamp device pixel ratio (do this first)

```js
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2))
```

A 3x phone renders **9x** the pixels of a 1x display. This single line routinely takes a
site from 20fps to 60fps on mobile. Clamp to 2; on heavy scenes clamp to 1.5.

Re-apply on resize — moving a window between monitors changes DPR.

---

## 2. Never read layout during scroll

Forced synchronous reflow is the classic scroll-site killer. These properties, read inside
a scroll handler, force the browser to re-layout the entire page **every event**:

`offsetTop` · `offsetHeight` · `getBoundingClientRect()` · `scrollHeight` · `clientWidth`
· `getComputedStyle()`

Measure once on load and on debounced resize, cache the numbers, read the cache during
scroll. ScrollTrigger already does this correctly — the danger is custom handlers written
alongside it.

---

## 3. One rAF loop

Covered in `SKILL.md`. Two loops = guaranteed micro-stutter. Audit for stray
`requestAnimationFrame(` and `setAnimationLoop(` calls.

---

## Asset budgets

| Asset | Budget | Notes |
|---|---|---|
| Total initial payload | < 3 MB | Above this, mobile users leave before first paint |
| Single GLTF | < 1.5 MB | Compress with Draco or Meshopt |
| Texture | 2048² max, 1024² preferred | Use KTX2/Basis for GPU-side compression |
| Draw calls | < 100 | Merge static geometry, use `InstancedMesh` for repeats |
| Triangles | < 500k | Most scroll sites need far fewer |

```bash
npx gltf-transform optimize in.glb out.glb --compress meshopt --texture-compress webp
```

KTX2 matters more than file size: a 2 MB PNG becomes ~8 MB of VRAM uncompressed, while
KTX2 stays compressed on the GPU. On memory-limited phones this is the difference between
running and crashing the tab.

---

## Load without blocking first paint

Render the page and a placeholder immediately; stream 3D in after.

```js
const manager = new THREE.LoadingManager()
manager.onProgress = (_, loaded, total) => setProgress(loaded / total)
manager.onLoad = () => {
  document.body.dataset.ready = 'true'
  ScrollTrigger.refresh()          // heights may have changed
}
```

Never gate the whole page behind a 3D loading screen unless the 3D *is* the content. A
scroll site should be readable and scrollable while the canvas is still empty.

---

## Reduced motion — mandatory, not optional

`prefers-reduced-motion` is a genuine accessibility need: parallax and scroll-jacking
trigger nausea and vertigo in people with vestibular disorders. Smooth scroll is exactly
the category of effect the setting exists for.

```js
const reduceMotion = matchMedia('(prefers-reduced-motion: reduce)')

function applyMotionPreference() {
  if (reduceMotion.matches) {
    lenis.destroy()                 // hand scrolling back to the OS
    ScrollTrigger.getAll().forEach((t) => t.kill())
    applyFinalStates()              // jump to end states — don't just freeze mid-animation
  }
}

applyMotionPreference()
reduceMotion.addEventListener('change', () => location.reload())
```

The key detail: **land on the end state, don't freeze the start state.** Killing triggers
without applying final values leaves content invisible if entrance animations began at
`opacity: 0`. Reduced motion must never mean reduced content.

---

## Other accessibility requirements

- **Content must exist in the DOM**, not only in the canvas. Text rendered as 3D geometry
  is invisible to screen readers and search engines. Keep real HTML behind/over the canvas.
- **`<canvas aria-hidden="true">`** when it's decorative — otherwise it's announced as an
  unlabelled graphic.
- **Keyboard navigation must work.** Verify Tab reaches every link and button. Scroll-jacking
  that traps keyboard focus is a hard accessibility failure.
- **Never hijack scroll distance.** If 1 wheel notch moves less than roughly 1 notch of
  content, users feel trapped. Pinning is fine; making scroll slow is not.
- **Respect `prefers-reduced-transparency`** and avoid depending on colour alone for state.

---

## Mobile specifics

```css
.section { height: 100dvh; }   /* not 100vh */
```

`100vh` on mobile is the viewport *without* browser chrome, so content sits under the URL
bar. `dvh` tracks the actual visible area.

Also on mobile:
- Leave `syncTouch: false` in Lenis (see `lenis-gsap-integration.md`)
- Ignore height-only resize events — they're the URL bar, not a real resize
- Test on a real mid-range Android, not just an iPhone or desktop emulation
- Consider skipping post-processing entirely below a width threshold

---

## Render-on-demand

If the scene is fully scroll-driven with no ambient motion, rendering 60fps while idle
burns battery for nothing.

```js
let needsRender = true
lenis.on('scroll', () => { needsRender = true })

gsap.ticker.add(() => {
  if (!needsRender) return
  renderer.render(scene, camera)
  needsRender = false
})
```

Only valid when nothing animates independently of scroll — no `uTime` shaders, no idle
rotation, no mixers. Otherwise render every frame.

---

## Pre-ship checklist

- [ ] DPR clamped to ≤ 2, re-applied on resize
- [ ] One rAF loop confirmed (no stray `setAnimationLoop` / `requestAnimationFrame`)
- [ ] `prefers-reduced-motion` path tested and lands on **visible** end states
- [ ] Tab key reaches every interactive element
- [ ] Real text in DOM; decorative canvas is `aria-hidden`
- [ ] `ScrollTrigger.refresh()` after assets load
- [ ] `invalidateOnRefresh: true` on all function-based start/end values
- [ ] Tested at 1280×800 and 375×812, and after a resize
- [ ] Console clean — no WebGL warnings, no context-lost errors
- [ ] Scrolled top→bottom **and bottom→top** (reverse breaks more often)
- [ ] Initial payload < 3 MB
- [ ] `dispose()` on teardown if the site is an SPA
