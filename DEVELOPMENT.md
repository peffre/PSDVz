# Development Notes

Handoff document for continued work on the point source field visualizer. The README covers
what the thing is and how to use it; this covers where it stands, why it's built the way it
is, and what's unfinished.

---

## Current state

Working and complete:

- 16-partial additive editor, drag-to-draw amplitudes, per-partial phase
- Delay-lookup pressure field, 1 / 2 / 4 sources
- Separation in wavelengths; per-source phase offsets for B, C, D; four quad phase presets
- **Source height above the render plane** — 3D distance, single global height for all sources
- Signed diverging colormap, adjustable `1/rᵃ` display falloff
- Particle displacement overlay (single source only, in-plane component only)
- Responsive layout: stacked below 900px, two-column with window-scaled field above
- Freeze/run

Two files carry the same behavior: `point-source-field.html` is the live one,
`point-source-field.jsx` is the earlier React version kept for reference. **The HTML is the
one to develop against.** The React version predates the height control and is now behind; if
it stops mattering, delete it rather than maintaining both.

### Code landmarks

Everything is one IIFE in a `<script>` at the bottom of the HTML.

| Symbol | Role |
| --- | --- |
| `S` | All mutable state. One plain object, read live by the render loop. |
| `rebuild()` | Recomputes `wave` and `disp` tables from `S.amps` / `S.phases`, then redraws the period trace and bars. Call this after any spectrum change. |
| `layout()` | Returns source positions as `{x, y, tag}` in normalized frame coords, `1.0` = half frame width. Note: no `z` — height is applied globally in `geometry()`. |
| `geometry()` | Builds one `Float32Array` distance field per source, `r = √(dx² + dy² + dz²)`. Cached against `geoKey`. |
| `dzPx` | Module-level. The current height in pixels, `(heightWL / wavelengths) · R`. Set by `geometry()`, read by the displacement overlay. |
| `gains()` | Builds `gLUT`, the `1/rᵃ` lookup indexed by normalized radius. Cached against `gainKey`. |
| `frame()` | The render loop. Inner pixel loop sums one delayed table read per source. Dimension-agnostic — it never learned about height. |
| `MAXPX` | Resolution ceiling per source count. |

The two caches matter: `geoKey` includes canvas size, source count, separation, wavelengths,
and now height; `gainKey` is just falloff. Anything that changes those triggers a rebuild of
several megabytes of typed arrays, so don't add anything to those keys casually.

---

## Design decisions worth not re-litigating

**No wave solver.** Air is non-dispersive, so `p = g(r)·s(t − r/c)` is exact for a free field,
not an approximation. Anyone arriving fresh will be tempted to reach for FDTD. Don't — it
would be slower, less accurate here, and would lose the property that arbitrary spectra cost
nothing. It's also the reason the 3D extension was cheap: the expression has no dimensionality
in it, so going 3D is a rendering question, never a physics one.

**Height is one scalar, not a plane basis.** Distance from a point source to any point on any
plane depends only on the perpendicular distance and in-plane position relative to the foot
point. An `{origin, u, v}` plane parameterization would be three vectors expressing what one
number already covers, and every extra degree of freedom would be a way to get lost. When
per-source heights arrive the parameterization changes, but it changes to a `z` per source,
not to a movable plane.

**Falloff is a display control, not physics.** Physically correct attenuation renders as a
bright dot surrounded by nothing *when the source is in the plane*. The exponent is exposed
precisely so the honest value is reachable but not mandatory. With height above zero the
singularity is gone and `a = 2.0` is genuinely readable — but the control stays, because
height zero is still the default and still useful.

**Signed colormap.** Absolute-value or magnitude coloring destroys the compression /
rarefaction asymmetry, which is the whole point — it's what makes odd versus even harmonic
content visible at a glance. This is also the thing most at risk in a future volumetric
version; see open thread 5.

**Separation in wavelengths — and height too.** Pixel-denominated distance becomes meaningless
the moment zoom changes. Wavelength units keep the interference geometry stable. `dzPx` is
computed with the same `/ S.wavelengths` normalization as `layout()` uses for `sepWL`; if one
changes, change both.

**Source A is the phase reference.** Offsets read as relative delays. An absolute phase per
source would be four sliders where three carry the same information.

**A lifted source draws as a crosshair.** A filled dot asserts "the source is here," which is
false the moment height is nonzero. This is not cosmetic — without it, the first conclusion
anyone draws from the stretched near-field rings is that the ring spacing is broken.

---

## Open threads

Roughly in order of value-to-effort. Items 1 and 2 are the same items they always were; 4
through 6 are the 3D roadmap that height opened up.

### 1. Multi-source particle displacement

Currently disabled for N > 1, and now also incomplete for N = 1: with the source lifted, the
overlay shows only the in-plane component, scaled by `ρ/r`. That factor is correct and the
resulting stillness at the foot point is real physics, but the out-of-plane component is
simply not drawn.

Doing it properly: displacement is a vector, so each dot sums the per-source contributions,
each directed along that source's radial. For dot at position **p**:

```
ξ(p) = Σ_s  disp[t − r_s/c] · g(r_s) · (p − s_s)/|p − s_s|
```

with **p** and **s** now 3-vectors and only the in-plane components of ξ drawn. Cost is N
table reads plus a normalize per dot, roughly 3000 dots — cheap. The reason to do it is that
**velocity nulls and pressure nulls do not coincide**, and showing both simultaneously in a
four-source field is something a pressure-only plot can't communicate. This is still the most
interesting unfinished item.

Once the dots are vector-driven, drop the polar grid in favor of a Cartesian one — the polar
arrangement only made sense when there was a single center in the plane, which height already
undermined.

Consider encoding the out-of-plane component as dot size or brightness rather than dropping it.

### 2. Per-source height

`geometry()` applies one `dzPx` to all sources. Returning a `z` from `layout()` and reading it
per source inside the loop is roughly two lines plus a UI decision.

What it unlocks is the case worth having: a raised pair or quad over a horizontal listening
plane, where the comb structure at ear height is *not* the comb structure in the plane of the
drivers. That's the practical loudspeaker question, and the current global height can't ask it.

UI is the harder half. Four height sliders is clutter; a small elevation-view thumbnail beside
the field, showing sources and plane in profile, would carry it better and would also make the
existing global height self-explanatory.

### 3. Per-source spectra

All sources currently share one waveform table. Giving each its own means:

- `wave` / `disp` become arrays of tables, one per source
- `rebuild()` runs per source
- the inner loop indexes `wave[s]` instead of `wave`
- the UI needs a source selector above the partial editor, with the bars editing the selected
  source's spectrum

This is what turns it from a physics demo into a routing tool: four different spectra at four
positions is a multichannel output configuration, and the field shows exactly where the
partials build comb structure.

### 4. Analytic null surfaces

For two sources, the locus of destructive interference at a given partial is the set where the
path difference is a half-integer number of wavelengths — a hyperboloid of revolution with the
sources as foci, one nested sheet per null order. Fully analytic: no sampling, no aliasing, no
solver, and cheap to draw as ordinary translucent geometry.

Currently the visualizer shows the intersection of those surfaces with the render plane, which
is what the dark spokes are. Drawing the surfaces themselves is the first thing that would make
this genuinely 3D rather than 3D-distance-on-a-2D-slice, and it explains directly why moving a
microphone vertically walks through the same comb a horizontal move does.

Four sources loses the clean conic form, but the pairwise families can still be drawn and
allowed to intersect.

### 5. GLSL volume raymarch

The real 3D version. The field is O(1) to evaluate anywhere, so a raymarcher is short:

```glsl
for (int i = 0; i < STEPS; i++) {
  vec3 p = ro + rd * (tNear + float(i) * ds);
  float v = 0.0;
  for (int s = 0; s < N; s++) {
    float r = length(p - src[s]);
    v += texture(waveTex, vec2(fract(t - off[s] - r * cycles), 0.5)).r * gain(r);
  }
  vec4 c = mapSigned(v);          // signed color, opacity from |v|
  acc.rgb += (1.0 - acc.a) * c.a * c.rgb;
  acc.a   += (1.0 - acc.a) * c.a;
  if (acc.a > 0.98) break;
}
```

Three things will bite, in order:

- **The signed colormap does not survive naive compositing.** Blending warm over cool along a
  ray averages toward ambient, so a rich field becomes brown fog — exactly the failure the
  signed map exists to prevent. Opacity must be a steep function of `|v|` (roughly `|v|³` with
  a dead zone near zero) so only strong compression or rarefaction is opaque and the
  near-ambient bulk is fully transparent. A max-intensity-projection toggle is worth having
  alongside it for reading peak structure.
- **Aliasing becomes the dominant problem.** Along a ray the radial rate is
  `dr/ds = dot(rd, normalize(p − src))`, so the waveform footprint per step varies from zero
  (ray tangent to a wavefront) to full (ray along the radial). Analytic and cheap to compute,
  but far too variable for a fixed box filter. Item 6 becomes a prerequisite rather than a
  nicety.
- **The geometry cache goes away.** A 3D distance field is hundreds of megabytes per source.
  Don't precompute it; `length()` on a GPU is free. `geoKey` disappears entirely, and with it
  the constraint above about not adding to it casually.

For Max this is `jit.gl.slab` with a custom `.jxs`, not `jit.gl.pix` — the latter won't give a
variable-trip-count loop. Sources, offsets, and height become uniforms; the waveform table
becomes a 1×2048 float texture writable from a `jit.matrix` or `buffer~`, which delivers item 7
almost for free. It would drop into a patch alongside the existing projector-controller and
light-show work.

The CPU version stays useful as the standalone slice tool and documentation artifact. It's
better for reading fine ring structure, freezing, and screenshots, and trying to make one file
do both jobs would compromise the layout logic and the `MAXPX` ceiling that already work.

### 6. Prefiltered waveform table

The 3-tap box filter in the inner loop is the cheapest thing that suppresses aliasing on the
upper partials. It's adequate at the current 16 partials and zoom range — and height only made
it safer, since off-plane the true radial rate `ρ/r` is below the `dr/dρ = 1` the filter
assumes, so the near-field plateau is over-filtered rather than aliased. It will still fall
apart if the partial count or zoom range is pushed, and it is not adequate for item 5.

Proper fix: build a mip pyramid over the waveform table at `rebuild()` time — successive
half-resolution copies — and select a level from the samples-per-pixel figure already computed
each frame as `h`. Trilinear between levels if the transitions show. Cost is a one-time build;
the inner loop gets *cheaper* because it becomes one tap instead of three.

### 7. Real signal input

Replace the additive table with a captured buffer — one period of an actual 3rd Wave output,
or any recorded waveform. The renderer doesn't care where the table comes from; only
`rebuild()` changes. Small change, large payoff for the wavetable case, since interpolation
artifacts would be directly visible in the ring structure.

### 8. Harmonic explode mode

Render each partial as its own ring layer before summing. Harmonic *n* has 1/n ring spacing,
so the nested-scale structure becomes visible, and dragging a partial's phase visibly rotates
its rings against the others. Discussed early, never built. Probably a small-multiples grid
rather than an overlay.

---

## Known limits

Not bugs — consequences of the model. Documented so they don't get "fixed" by accident.

- **Free field only.** No boundaries, reflections, absorption, or room. The field assumes the
  source has been running forever; there is no onset transient.
- **No sound engine.** Oscillation rate is nominal; frequency is normalized, not in Hz.
  Adding audio would mean deciding what the visual `c` corresponds to physically, which is
  currently a free parameter chosen for legibility.
- **One height for all sources.** See open thread 2.
- **Displacement shows the in-plane component only,** and only for a single source.
- **The render plane is fixed in orientation.** Correct and complete for one source; genuinely
  limiting only once per-source heights exist.
- **Sources can leave the frame** at large separation with low wavelength counts. Deliberately
  unclamped — clamping would make the λ readout lie. Raising `Wavelengths` brings them back.
  Height has no such failure mode; a source above the plane is off-frame in a direction the
  frame doesn't have.

### Things to watch

- Four sources at a large window is the worst case: `pixels × 4` table reads per frame. The
  `MAXPX` ceiling handles it, but any added per-pixel work multiplies by source count.
- `geometry()` allocates fresh `Float32Array`s on every rebuild. At 1000px × 4 sources that's
  16MB churned per geometry change. The height slider now triggers this on every drag step —
  it's the most rebuild-heavy control in the interface. Reusing the buffers, or debouncing
  during drag, would be worth doing before adding per-source heights.
- `dzPx` is written by `geometry()` and read by the displacement overlay in `frame()`. That
  ordering is currently guaranteed because `frame()` calls `geometry()` first. Anything that
  reads `dzPx` outside the loop needs to respect that, or compute it directly from `S`.
- The `resize` listener resets `geoKey` and redraws the period trace. Anything else that
  depends on layout needs adding there.
