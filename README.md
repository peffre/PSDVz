# Point Source — Compression / Rarefaction Field

An interactive visualization of acoustic pressure radiating from one, two, or four point
sources, driven by a 16-partial additive spectrum editor.

Compression is warm, rarefaction is cool, ambient pressure is the neutral background. The
concentric ring structure you see **is** the source waveform, drawn outward and reversed in
time — change the spectrum and the rings change shape immediately.

Sources may sit above the render plane rather than in it, which turns the flat slice into a
cross-section of a 3D field.

Single self-contained HTML file. No build step, no dependencies, no network access. Open
`point-source-field.html` in a browser, or point a `jweb` object at it to embed it in a Max
patch.

---

## Method

The visualization does not solve the wave equation. It doesn't need to.

Air is non-dispersive: every partial travels at the same speed `c`, so the waveform shape is
preserved as the wavefront expands. Pressure at any point is therefore just the source signal
read at a delay, scaled by distance:

```
p(x, y, t) = g(r) · s(t − r/c)        r = distance from the source
```

That reduces to one table lookup per pixel per source. Consequences worth knowing:

- **No stability constraint.** There's no CFL condition, no grid dispersion, no numerical
  damping. Nothing to tune for the solver's sake.
- **Arbitrary spectra are free.** Inharmonic partials, transients, and discontinuities all
  work, because the code reads an actual signal rather than summing a truncated Fourier
  series at render time.
- **Multiple sources are a sum.** Interference is exact, not emergent from a discretization.
- **Dimension is free too.** Nothing in the expression cares whether `r` is a 2D or 3D
  distance, which is why lifting a source out of the render plane costs one term in a square
  root and no change at all to the render loop.
- **It's a snapshot of steady state.** There are no boundaries, no reflections, and no
  onset — the field assumes the source has been running forever. Room acoustics are out of
  scope by construction.

The source waveform is built once into a 2048-sample table (`LUT`) whenever the spectrum
changes, along with its running integral, which gives particle displacement.

### Aliasing

At high `Wavelengths` settings, the upper partials approach one cycle per pixel and would
alias into moiré. Each lookup is a 3-tap box filter whose width tracks one pixel of radius
(`h` in the render loop), which keeps partial 16 clean across the full zoom range. It's the
cheapest thing that works; a proper mip pyramid over the waveform table would be better if
you push the partial count higher.

The filter assumes `dr/dρ = 1`, which is true only in the plane of the source. Off-plane the
true rate is `ρ/r ≤ 1`, so raising `Height` makes the filter conservative rather than
insufficient — the near-field plateau is slightly over-filtered and never aliases.

---

## Controls

### Harmonic partials

Drag across the bars to draw the spectrum — sixteen partials, amplitude only. The selected
partial's phase gets its own slider below.

Phase is worth playing with: sweep partial 3 or 5 and watch the ring *shape* change while the
magnitude spectrum stays identical. Presets are the usual set — Sine, Saw, Square, Triangle,
Pulse (all partials equal), Odd — plus Clear, which returns to a bare fundamental.

The **Period** trace to the right shows one cycle of the resulting waveform. Everything in the
field is derived from that trace.

### Sources

| Setting | Layout |
| --- | --- |
| One | Centered |
| Two | Horizontal pair, A left / B right |
| Four | Square, A/B on top, C/D below |

`Separation` is measured **in wavelengths**, not pixels, so the interference geometry stays
meaningful when you change the zoom. For four sources it's the side length of the square.

Source A is always the phase reference at 0°; B, C, and D get their own offset sliders, so the
numbers read as relative delays. The four-source quad presets:

- **In phase** — center antinode with a null grid around it.
- **Alternating** — diagonal pairs opposed. The center goes permanently null and energy
  squeezes out along the diagonals.
- **Rotating** — 0 / 90 / 270 / 180 around the square. The field circulates rather than
  standing. With a rich spectrum each partial appears to rotate at a different rate, because
  the offset is fixed in time rather than in cycles.
- **Pair flip** — front pair against rear pair; the dipole figure-eight.

The radiating dark spokes are comb nulls. Each partial nulls at a different angle, which is
the visual reason a comb filter sounds like a comb rather than a hole.

### Field

- **Wavelengths** — how many wavelengths span the radius of the frame. This is the zoom.
- **Falloff 1/rᵃ** — display gamma on the distance attenuation, not physics. `a = 2.0` is
  honest inverse-square and, with the source in the plane, mostly unreadable: you get a bright
  dot and little else. Around 0.5 keeps the outer wavefronts visible. Raising `Height` removes
  the singularity and makes the honest value usable.
- **Height above plane** — perpendicular distance from the source to the render plane, in
  wavelengths. See below.
- **Particle displacement** — overlays a polar dot field driven by the integral of pressure.
  Dots bunch at the color *transitions*, not the color peaks, because displacement and
  pressure are 90° apart. Single-source only; the overlay assumes one radial center, and doing
  it properly for multiple sources means summing vector displacements per dot.
- **Freeze / Run** — stops time. Useful for reading fine ring structure or for screenshots.

---

## Height above the plane

At `Height = 0` the source lies in the render plane and the rings are evenly spaced — one per
wavelength, out to the frame edge. Lift the source and the in-plane distance becomes

```
r(ρ) = √(ρ² + d²)
```

where ρ is the in-plane radius from the foot of the perpendicular and `d` is the height. Ring
spacing follows `dr/dρ = ρ/r`, so it stretches directly under the source and tightens toward
the true wavelength far out. The broad flat plateau over the source is the geometric near
field: the wavefront there is nearly parallel to the plane, so moving sideways barely changes
your delay. It's the reason a ceiling speaker sounds different directly underneath than three
meters away — the driver hasn't changed, the geometry has.

**Plane orientation is not a control, on purpose.** Distance from a point source to any point
on any plane depends only on the perpendicular distance and the in-plane position relative to
the foot point. Tilting the plane about the source, or sliding it sideways, only reframes the
same picture. One scalar covers the whole family. (This stops being true for multiple sources
at different heights, which is a future control, not a present one.)

Two consequences worth noticing:

- **Inverse-square becomes legible.** With `d > 0` the field never approaches the singularity,
  so `Falloff = 2.0` renders as a real image rather than a bright dot. The internal `r0` clamp
  in `gains()` stops doing any work.
- **The displacement dots go still at the center.** The motion is along the 3D radial and only
  its in-plane projection is visible on the slice, a factor of `ρ/r`. Directly under a raised
  source the air is moving straight at the viewer. The rings keep travelling through a
  stationary dot field, which is correct and slightly startling.

A lifted source draws as a hollow crosshair with a `↑ d λ` readout rather than a filled dot,
because it isn't in this plane and a solid marker would say otherwise.

For two and four sources the whole array is lifted together, parallel to the slice — the
ceiling-array case. Per-source heights are not yet exposed.

---

## Layout and performance

Below 900px viewport width the interface is a single stacked column. At 900px and above it
switches to a two-column grid: the field scales to fill the window height, and the controls
move to a scrolling side rail. The page itself doesn't scroll on desktop, so the field stays
fixed while you work.

Render cost is roughly `pixels × sources`, so the internal resolution ceiling steps down as
sources are added:

```js
const MAXPX = {1: 1000, 2: 820, 4: 660};
```

Raise these if your machine has headroom — the four-source field is upsampled slightly by CSS
at large window sizes, which is barely visible given how smooth the field is.

Two things keep the loop fast: the `1/rᵃ` gain is a precomputed lookup indexed by normalized
radius rather than a `Math.pow` per tap, and the per-source distance fields are cached and
rebuilt only when the geometry actually changes. Moving the falloff or wavelength sliders
triggers no rebuild at all. `Height` does trigger one, since it changes every distance.

---

## Extending it

State lives in one plain object, `S`, and the render loop reads it live. A few entry points:

- **Drive the spectrum externally.** Write into `S.amps` / `S.phases` and call `rebuild()`.
  That's the whole contract — hook it to analysis data, MIDI, or OSC.
- **Add a fifth source.** `layout()` returns an array of `{x, y, tag}` in normalized frame
  coordinates where `1.0` is half the frame width. The renderer is generalized over that list;
  extend the array and add an entry to `S.off`.
- **Per-source height.** `geometry()` currently applies one `dzPx` to every source. Returning a
  `z` from `layout()` and reading it inside the source loop is a two-line change, and it's what
  makes a raised stereo pair over a horizontal listening plane possible.
- **Change the colormap.** `buildPalette()` builds a 512-entry diverging ramp. Keep it signed —
  taking the absolute value destroys the compression/rarefaction asymmetry that makes odd
  versus even harmonic content visible in the first place.
- **Per-source spectra.** Currently all sources share one waveform table. Giving each its own
  table means one `rebuild()` per source and indexing `wave` by source in the inner loop.

### Known limits

- No sound engine. Oscillation is a fixed nominal rate; frequency is normalized rather than
  in Hz.
- Free field only — no boundaries, reflections, or absorption.
- The rendered slice is a 2D cross-section of a 3D field. Falloff is an exposed exponent
  rather than a fixed `1/r`, so the physically honest value is reachable but not mandatory.
- Particle displacement is single-source only, and shows only the component lying in the plane.
- All sources share one height.

---

## Files

```
point-source-field.html   the whole thing
point-source-field.jsx    earlier React version, predates the height control
README.md
DEVELOPMENT.md
```
