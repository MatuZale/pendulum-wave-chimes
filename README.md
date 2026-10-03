# Pendulum Wave Chimes

A generative ambient instrument. A row of pendulums is tuned so that each one swings exactly
one more time per cycle than the one before it. Every time an arm crosses the centre line a
bell sounds. Because the swing counts are consecutive integers, the arms drift out of phase,
pass through travelling-wave and grouped states, and re-align only when the cycle closes — so
the music never repeats inside a cycle, and with the right settings not for several cycles.

**Live: https://matuzale.github.io/pendulum-wave-chimes/**

Single HTML file. No build step, no dependencies, no network calls except two Google Fonts
stylesheets. Open `index.html` in any modern browser and press Play. Audio needs a user
gesture, so nothing sounds until you do.

Served straight from the repository root by GitHub Pages — there is nothing to build, so the
file you read here is byte-for-byte the file the page serves.

---

## 1. The physics

### 1.1 Crossing times

Each arm is released at maximum deflection at `t = 0`, so under the small-angle approximation

```
θ_k(t) = A · cos(2π t / P_k)
```

Centre crossings are the zeros of the cosine, which fall a quarter period in and then every
half period:

```
t = P_k (2j + 1) / 4        j = 0, 1, 2, …
```

Arm `k` is tuned to complete `n_k = base + k` swings per cycle `T`, so `P_k = T / n_k` and the
period cancels out. With a stride `s` (the Density control — sound every `s`-th crossing):

```
t_{k,m} = T (2 m s + 1) / (4 n_k)
```

That expression is the whole timing model. Everything else — the visual flash, the ripples,
the strike axis — is derived from it. The amplitude `A` never appears, because in the
small-angle regime the period is independent of amplitude (isochronism), which is why the
Breathing control can change loudness without touching the rhythm.

### 1.2 Frequency drift

Consecutive integer swing counts alone still repeat every cycle or two. To push the horizon
out, each arm's instantaneous frequency is slowly modulated, with its own phase offset:

```
f_k(t) = (n_k / T) · (1 + d · sin(2π t / Td + ψ_k))
```

A varying period has no closed-form crossing time, but the phase does. Integrating,

```
φ_k(t) = 2π (n_k / T) · W(t, ψ_k)
W(t, ψ) = t + d · (Td / 2π) · (cos ψ − cos(2π t / Td + ψ))
```

so each arm behaves exactly as before, but in **warped time** `W`. Crossings are the §1.1
formula evaluated in `W`, mapped back to real time by inverting it. That is safe because
`dW/dt = 1 + d·sin(…) > 0` for `d < 1`, so `W` is strictly monotonic. The inverse is one
fixed-point guess plus two Newton steps; measured residual ~1e-14 s.

Two properties make this worth the trouble:

- `W(0) = 0` for every `ψ`, so the rig still starts perfectly in phase.
- `W(m·Td) = m·Td`, so warped time rejoins real time every drift period.

The second one keeps the repeat period exact rather than destroying it. Arm `k`'s crossing set
is invariant under both `q_k·T` and `Td`, so with `Td = c·T` the rig repeats after
`lcm(lcm(q_k), c)` cycles — the drift *multiplies* the horizon instead of blurring it.
Cathedral goes from 2:24 to 12:00, Tide from 8:00 to 40:00, Vigil to 70 minutes.

`ψ_k` is spread across the arms by the Sequence control: at 0 they all speed up and slow down
together; at 100 the change travels along the rig one arm at a time.

Because frequency really is changing, so is length (`L ∝ 1/f²`), and the drawn radii follow it
(`radius ∝ L^0.6 ∝ ρ^-1.2`). The fan visibly breathes in and out. The lengths in the table are
nominal — the mid-point of that breathing.

### 1.3 Scatter: release angle and detune

Two per-arm randomisations, drawn from a seeded LCG so a configuration is reproducible and
Reseed genuinely redraws it.

**Release angle.** Arm `k` starts at its own angle `α_k`, so the ladder of crossings slides:

```
θ_k(t) = A · cos(2π W(t,ψ_k) / P_k + α_k)
W_{k,j} = P_k · ( (2j+1)/4 − α_k / 2π )
```

Still closed-form. The only new bookkeeping is a floor on the index, since a large `α` pushes
the first crossing to a negative time: `m_min = ⌈ (2α/π − 1) / 2s ⌉`.

**Detune.** The swing count itself stops being an integer: `n_k → n_k·(1 + ε_k)`. That is what
finally removes any exact repeat — with irrational ratios no common multiple exists, so the
readout says so rather than printing a number that is no longer true. Setting Detune to zero
restores the integer ratios and the exact repeat period.

The control caps at 1.5 % because neighbouring arms must not swap order. The gap between `n`
and `n+1` is `100/n` percent, so the safe bound is `50/n`:

| `n` | gap to next arm | detune must stay under |
|---|---|---|
| 12 | 8.33 % | 4.17 % |
| 20 | 5.00 % | 2.50 % |
| 30 | 3.33 % | 1.67 % |
| 43 | 2.33 % | 1.16 % |

Layout takes max/min over the radius array rather than assuming index 0 is the longest arm, so
even a swap degrades gracefully.

### 1.4 Arm lengths

Radii and the "Bill of materials" table use the real length of a simple pendulum:

```
L = g T² / 4π²          g = 9.80665 m/s²
```

These get large fast. A 4.5 s period is a 5 m arm; a 10 s period is a 25 m arm, which is
Foucault-pendulum territory. At the 30-minute end of the Cycle slider you are asking for
kilometre-scale arms. The numbers are honest, not scaled for comfort.

In the visualisation, radius is proportional to `L^0.6`. The raw `L ∝ 1/n²` law crowds the
short arms into a thin band; the 0.6 exponent keeps the ordering and keeps the bass arms
visibly the long ones while spreading the short end out.

### 1.5 How long before it repeats

The *shape* returns to its start every cycle, because `cos(2π n_k) = 1` for integer `n_k`.
The *rhythm* need not. Arm `k` returns to its own strike sequence after

```
q_k = s / gcd(s, 2 n_k)     cycles
```

and the whole rig after `lcm(q_k)` cycles. Stride 1 and 2 always divide `2n`, so those repeat
every cycle. Stride 4 splits odd and even arms (`q = 2`); stride 8 splits them four ways.

| Stride | Arms affected | Rig repeats after |
|---|---|---|
| 1, 2 | none | `T` |
| 4 | odd `n` | `2T` |
| 8 | depends on `n mod 4` | up to `4T` |

On top of that the Breathing envelope runs on `T·φ` (golden ratio). Irrational ratio to the
cycle means the loud moments never land on the same part of the pattern twice, so even after
the strike pattern repeats the piece sounds different.

---

## 2. Sound

### 2.1 Scheduling

Nothing is triggered from the animation frame. `requestAnimationFrame` jitters by several
milliseconds and drops frames; timing the polyrhythm off it would make it drift audibly.
Instead a `setInterval` pump (140 ms tick, 1.2 s look-ahead) computes crossing times
analytically and schedules them at exact `AudioContext` times.

The simulation clock is derived from `actx.currentTime`, not `performance.now()`, so the
picture and the sound read the same hardware clock and cannot diverge.

### 2.2 Signal graph

```
strike voices ─┬─────────────────────────────► dry gain ─┐
               └─► pre-delay 45 ms ─► convolver ─► LP 3.2k ─► wet gain ─┤
drone ─► LP 260 Hz ─┬────────────────────────────────────────────────────┤
                    └─► send 0.16 ─► pre-delay ─┘                        │
chord pad ─► LP 900 Hz ─┬────────────────────────────────────────────────┤
                        └─► send 0.9 ─► pre-delay ─┘                     │
                                                                          ▼
                                        pre-comp ─► HP 32 Hz ─► compressor ─► tilt shelf ─► master
```

The reverb impulse response is generated at runtime: decaying white noise, 7.5 s, envelope
`(1−t)^2.1`, with a short fade-in to avoid a click at sample zero.

### 2.3 Voices

**Chime** — additive, with the inharmonic partials a struck bar actually has, each on its own
envelope, shorter for the higher partials:

| Ratio | Level | Duration |
|---|---|---|
| 1.00 | 1.00 | full |
| 2.00 | 0.20 | 0.55× |
| 3.93 | 0.09 + 0.16·tone | 0.24× |
| 9.20 | 0.008 + 0.05·tone | 0.07× |

Plus a band-passed noise burst for mallet contact, which fades out entirely at dark tone
settings.

**Deep bass** — sub-octave (only above 42 Hz, see §5.2), fundamental, 2.01 and 2.98, longer
attack, low-passed, up to 9 s.

Both use a **two-stage decay**: a fast drop to 30 % over the first 11 % of the duration, then
the long tail. A single exponential is a constant dB/s slide, which reads as a held sine
rather than a struck bell.

**Drone** — three sines at 1, 2^(7/12) and 2 above the lowest bell, low-passed at 260 Hz, with
a slow shared amplitude LFO. Equal-tempered ratios, no frequency drift — see §5.3.

**Chord pad** — two banks of three oscillators. A chord change writes the new frequencies into
the silent bank and crossfades, so the oscillators never stop and nothing clicks. Chords are
stacked thirds taken from the active scale, so they cannot fall outside the key. The
progression has six degrees against four segments per cycle, so it only comes back round every
third cycle.

### 2.4 Loudness

One law for everything:

```
amplitude = 0.40 / √(density · (0.6 + 0.5 · sustain))
```

with density = `2 Σn / (T·s)` strikes per second. Raise the density or lengthen the sustain and
each individual bell gets quieter, so the sum stays put. The drone and pad are derived from the
same figure, which is what keeps them in proportion instead of drowning the bells whenever the
texture gets busy.

---

## 3. Picture

Three stacked canvases:

1. **Sky** — starfield plus three nebula clouds in the palette colours, drawn once per resize
   from a seeded LCG so a resize does not reshuffle it. Parallax is a CSS transform, handled by
   the compositor, so it costs nothing per frame.
2. **Bodies** — pre-rendered radial-gradient sprites drawn with `globalCompositeOperation =
   "lighter"`, so overlapping glows genuinely add.
3. **Overlay** — strike axis, ripples, brackets, pivot. Cleared every frame.

**The visual envelope is the audio envelope.** `voiceDur(p)` returns the same duration the
synthesiser uses for that arm, and `ampEnv(age, dur)` is the same two-stage curve. A bell you
hear ringing is a bob you see ringing, not an approximation of one. The same applies to the
ripples leaving the strike axis and to the global brightness, which follows the `T·φ` swell.

**Telling the two families apart** — one rig, four independent signals:

- a clear radial gap (19 % of the span) between the last bass arm and the first chime
- a size step: pitch ramp × 1.45 for bass, × 0.82 for chimes
- the colour ramp walked in two runs with a hole between them, so the groups share no shades
- a rim circle on every bass bob, like the lip of a bell

Plus labelled brackets outside the swing range.

---

## 4. Controls

**Voice** — five presets. Each is checked against its strike density and true repeat period.

| Voice | Arms | Cycle | Density | Drift | Spread | Detune | Repeat (detune off) |
|---|---|---|---|---|---|---|---|
| Cathedral | 12 | 1:12 | 1.79/s | 1.2 % / 5 cyc | 28 | 0.30 % | 12:00 |
| Nocturne | 14 | 1:24 | 2.04/s | 1.6 % / 7 cyc | 46 | 0.45 % | 19:36 |
| Tide | 9 | 2:00 | 0.45/s | 0.8 % / 5 cyc | 18 | 0.20 % | 40:00 |
| Aurora | 16 | 1:36 | 2.46/s | 2.2 % / 3 cyc | 60 | 0.70 % | 9:36 |
| Vigil | 14 | 5:00 | 0.85/s | 1.0 % / 7 cyc | 34 | 0.35 % | 70:00 |
| Aleatoric | 15 | 1:20 | 2.4/s | 2.6 % / 4 cyc | 100 | 1.30 % | — |

**Sound** — Volume; Reverb (dry room → cathedral); Tone (dark → bright, drives the partial
levels, the voice low-pass and a high-shelf tilt); Sustain (0.4× – 2.6×); Harmony (drone +
chord pad); Breathing (the `T·φ` swell).

**Density** — Full / Half / Quarter / Eighth. The single most effective calm control, and it
stretches the repeat period as a side effect.

**Scatter** — Release spread (how far the starting angles are scattered); Detune (0–1.5 %, how
far the periods wander off the exact ratios); Reseed (draw a new instrument).

**Drift** — Depth (0–5 %, how far each swing stretches); Period (2–12 cycles, and the factor
the repeat horizon is multiplied by); Sequence (all together → one arm at a time).

**The rig** — Pendulums (4–28); Cycle length (24 s – 30 min, logarithmic: `24 · 75^(v/1000)`,
because a linear slider makes the short end unusable); Swings per cycle for the longest arm.

**Tuning** — Key; Register (Sub / Low / Mid / High); Scale (Bell, Open fifths, Suspended,
Hirajoshi, Pentatonic minor/major, Yo); Deep-bass arm count.

**Visuals** — Glow; Starfield; Orbital drift (the rig sways 3.5° on a `T·2.618` period, so the
motion never looks mechanical); Arms and arcs.

Space bar or a click on the canvas toggles play. In fullscreen the interface fades out after
2.8 s without mouse movement.

---

## 5. Debugging log

Four real bugs, kept here because each one has a lesson that outlives this project.

### 5.1 Aliasing from an unbounded pitch ladder

The pitch ladder was `root + 12·⌊k/L⌋ + steps[k mod L]`. With the two-note Open fifths scale
and 28 arms that climbs **fourteen octaves**: arm 21 landed on a 50 kHz fundamental and arm 28
on 601 kHz, with a 9.2× partial at 5.5 MHz. Everything above the 24 kHz Nyquist frequency
folded back as noise at unrelated pitches.

Fix: build the ladder from the scale pitches inside a capped span (54 semitones, hard ceiling
at MIDI 93 ≈ 1760 Hz) and spread the arms evenly over that pool. When there are more arms than
pitches, several share a note — which, at different periods, is three independent rhythms on
one bell. Presets whose natural span is under the cap are untouched.

### 5.2 Sub-audio content pumping the compressor

The bass voice added a sub-octave unconditionally. On low presets that put energy at 24.5 and
36.7 Hz — below the pitch threshold, audible only as flutter. Worse, the compressor's 6 ms
attack is shorter than a 30 Hz cycle (33 ms), so it tracked individual waveform cycles instead
of the envelope and turned gain reduction into audible pumping.

Fix: sub-octave only when it lands at 42 Hz or above; a 32 Hz high-pass before the compressor;
attack lengthened to 18 ms.

### 5.3 A just fifth against equal-tempered bells

The drone's fifth partial used the just ratio 1.5. The bells are equal-tempered, where a fifth
is 2^(7/12) = 1.4983. The 0.113 % difference is nothing on paper and a **0.083 Hz beat** at
these registers — a twelve-second throb under the whole piece, heard as "it resonates instead
of fading".

| Preset | Drone 1.5× | Nearest bell | Beat |
|---|---|---|---|
| Tide | 73.499 Hz | 73.416 Hz | 0.083 Hz → 12.1 s |
| Cathedral | 82.500 Hz | 82.407 Hz | 0.093 Hz → 10.7 s |
| Nocturne | 55.062 Hz | 55.000 Hz | 0.062 Hz → 16.1 s |

The first fix made it worse. Switching to the equal-tempered ratio and adding 6 cents of slow
drift "to animate it" simply re-created the beat at a different rate. Measured over a minute
against a bell on the same pitch:

```
just 1.5 (before)        12.95×  peak/trough
ET fifth + 0.00 cents     1.01×
ET fifth + 0.25 cents     1.06×
ET fifth + 1.00 cents     2.18×
ET fifth + 2.00 cents    14.36×
```

So: exact equal temperament, zero drift. The drone locks phase with a coincident bell rather
than beating against it, which is inaudible. Rendering the real graph in an
`OfflineAudioContext` confirmed 2.55× → 1.45×, the remainder being the intended 3 dB swell.
The same reasoning removed a per-strike random detune added for the same bad reason.

### 5.4 Permanent trail residue

Motion trails were made by fading the layer toward transparency with `destination-out` at low
alpha. That works in continuous maths and not in 8 bits: once the per-frame alpha step rounds
below 1/255 the pixel stops fading and the smear is permanent.

Fix: clear every frame and compute the tail analytically instead — six samples backwards along
`θ(t − s·19 ms)` with decreasing alpha. It is a shutter rather than a memory, so it cannot
leave anything behind, and it is physically correct: fast-moving bobs stretch, bobs pausing at
the turning point stay round.

---

## 6. Not done

- **Offline render.** `OfflineAudioContext` could render an arbitrarily long track to a WAV
  faster than real time — the scheduler is already time-parameterised, so it mostly needs a
  render loop and a WAV encoder.
- **AudioWorklet.** At Full density with many arms and long sustain the graph carries a few
  hundred oscillator nodes. A worklet running one additive engine would be far cheaper.
- **Preset sharing.** The whole state is a small flat object; URL-hash serialisation would make
  settings shareable.

---

## 7. Files

```
pendulum-wave-chimes/
├── index.html    the entire instrument
└── README.md     this file
```

Roughly 1300 lines: ~460 CSS, ~750 JavaScript, the rest markup.
