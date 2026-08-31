# Scroll Pattern Cookbook

All patterns assume the single-ticker setup from `SKILL.md` and a shared `state` proxy
object. Scroll writes to `state`; the render loop reads it.

---

## 1. Camera along a path

The signature "flythrough" effect. Define a curve, sample it by progress.

```js
const curve = new THREE.CatmullRomCurve3([
  new THREE.Vector3(0, 2, 10),
  new THREE.Vector3(5, 1, 4),
  new THREE.Vector3(2, 4, -3),
  new THREE.Vector3(0, 1, -8),
])

const lookTarget = new THREE.Vector3()

function render() {
  const p = THREE.MathUtils.clamp(state.progress, 0, 1)
  curve.getPointAt(p, camera.position)            // arc-length parametrised = even speed
  curve.getPointAt(Math.min(p + 0.02, 1), lookTarget) // look slightly ahead
  camera.lookAt(lookTarget)
}
```

Use `getPointAt` (arc-length, uniform speed), **not** `getPoint` (raw parameter, speeds
up and slows down unevenly between control points). Pass the target vector to avoid
allocating a new `Vector3` every frame.

---

## 2. Pinned section

Hold a section still while scroll drives the 3D. `end: '+=200%'` means "pin for two
viewport heights of scrolling."

```js
ScrollTrigger.create({
  trigger: '.pin-section',
  start: 'top top',
  end: '+=200%',
  pin: true,
  pinSpacing: true,
  scrub: 1,
  onUpdate: (self) => { state.progress = self.progress },
})
```

`pinSpacing: true` inserts padding so following content isn't overlapped. Set it to
`false` only when you want the next section to slide over the pinned one.

---

## 3. Scrubbing a baked GLTF animation

For a model with a keyframed animation you want tied to scroll rather than played.

```js
const mixer = new THREE.AnimationMixer(model)
const clip = gltf.animations[0]
const action = mixer.clipAction(clip)
action.play()
action.paused = true          // critical — we set time manually

function render() {
  action.time = clip.duration * THREE.MathUtils.clamp(state.progress, 0, 1)
  mixer.update(0)             // 0 delta: apply the time we just set, don't advance
}
```

Forgetting `action.paused = true` makes the animation play *and* get scrubbed, which
looks like stuttering.

---

## 4. Progress-driven shader uniform

Three.js uniforms are `{ value: x }` objects, so GSAP can tween `.value` directly — no
proxy needed.

```js
const material = new THREE.ShaderMaterial({
  uniforms: { uProgress: { value: 0 }, uTime: { value: 0 } },
  vertexShader, fragmentShader,
})

gsap.to(material.uniforms.uProgress, {
  value: 1,
  ease: 'none',
  scrollTrigger: { trigger: '#reveal', start: 'top bottom', end: 'bottom top', scrub: 1 },
})
```

Keep `uTime` on the render loop (continuous) and `uProgress` on scroll (discrete). Mixing
them into one uniform makes the effect impossible to tune.

---

## 5. Scroll velocity → distortion

The "stretchy on fast scroll" effect. Lenis emits velocity; smooth it before use or it
looks twitchy.

```js
lenis.on('scroll', ({ velocity }) => { state.rawVelocity = velocity })

function render() {
  // exponential smoothing — raw velocity is far too spiky to use directly
  state.velocity += (state.rawVelocity - state.velocity) * 0.1
  material.uniforms.uDistort.value = THREE.MathUtils.clamp(state.velocity * 0.02, -1, 1)
}
```

The Lenis scroll event gives `{ scroll, limit, velocity, direction, progress }`.

---

## 6. Horizontal scroll section

```js
const track = document.querySelector('.h-track')

gsap.to(track, {
  x: () => -(track.scrollWidth - window.innerWidth),
  ease: 'none',
  scrollTrigger: {
    trigger: '.h-scroll',
    start: 'top top',
    end: () => '+=' + (track.scrollWidth - window.innerWidth),
    pin: true,
    scrub: 1,
    invalidateOnRefresh: true,   // recompute the function values on resize
  },
})
```

`invalidateOnRefresh: true` is mandatory whenever `end` or `x` are functions — without it
the distances are frozen at first paint and break on resize or orientation change.

---

## 7. Section-based scene state

For discrete changes (swap model, change palette) rather than continuous scrubbing.

```js
const sections = gsap.utils.toArray('[data-scene]')

sections.forEach((el, i) => {
  ScrollTrigger.create({
    trigger: el,
    start: 'top center',
    end: 'bottom center',
    onEnter:     () => activateScene(i),
    onEnterBack: () => activateScene(i),
  })
})
```

Both `onEnter` and `onEnterBack` are needed or scrolling up leaves the wrong scene active.

---

## 8. Parallax layers

```js
const layers = [bgGroup, midGroup, fgGroup]
const depths = [0.2, 0.5, 1.0]

function render() {
  layers.forEach((layer, i) => {
    layer.position.y = state.progress * 6 * depths[i]
  })
}
```

Depth values should be non-linear and hand-tuned. Evenly spaced multipliers read as flat.

---

## 9. Mouse parallax on top of scroll

Layer subtle pointer movement over scroll-driven camera motion — do it as an *offset*, so
it never fights the scroll path.

```js
const pointer = { x: 0, y: 0, tx: 0, ty: 0 }
window.addEventListener('pointermove', (e) => {
  pointer.tx = (e.clientX / window.innerWidth  - 0.5) * 2
  pointer.ty = (e.clientY / window.innerHeight - 0.5) * 2
})

function render() {
  pointer.x += (pointer.tx - pointer.x) * 0.05   // lerp, never assign directly
  pointer.y += (pointer.ty - pointer.y) * 0.05
  curve.getPointAt(p, camera.position)
  camera.position.x += pointer.x * 0.3           // offset after the path sample
  camera.position.y += -pointer.y * 0.2
  camera.lookAt(lookTarget)
}
```

---

## 10. Reveal on enter (non-scrubbed)

For one-shot entrance animations, drop `scrub` entirely and use `once`.

```js
gsap.from('.card', {
  y: 60, opacity: 0, duration: 0.8, stagger: 0.1, ease: 'power3.out',
  scrollTrigger: { trigger: '.cards', start: 'top 75%', once: true },
})
```

Scrubbing entrance animations is a common mistake — it makes content flicker in and out
as users scroll up and down. Entrances should fire once.
