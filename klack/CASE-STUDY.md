# Klack — building a mechanical keyboard sound engine for macOS

**What it is:** a macOS menu-bar app that plays mechanical-keyboard click sounds on every
keystroke, with a custom DSP synthesis engine, a live mixer, an import pipeline for real
recordings, and a skeuomorphic control panel drawn entirely in Core Graphics.

**Stack:** Swift / AppKit / AVFoundation / Core Graphics, plus Python for offline audio
generation. No third-party dependencies. Roughly 2,500 lines of Swift and 550 of Python.

**Context:** built collaboratively with Claude in a single working session. This document covers
the decisions and the mistakes, including the ones that were wrong at first.

---

## 1. The premise

The category already exists — there are several apps that make your laptop sound like a
mechanical keyboard. Nearly all of them are sample players: someone recorded a nice board, and
the app fires those WAV files back at you.

That approach has a ceiling. You get exactly the keyboards the developer recorded. You can't dial
in "the same switch but less bright," because brightness isn't a parameter of a WAV file — it's
baked into the recording.

So the goal was a step past sample playback: **synthesize the click**, and expose the synthesis
parameters as a mixer. If the sound is generated from a model, "less bright" is a number you can
turn.

That produced two constraints that shaped everything after:

1. **The synthesized click has to not sound synthetic.** This is the whole ballgame. A click that
   reads as a computer beep is worse than a mediocre sample.
2. **The mixer has to be live.** If dragging a knob takes half a second to produce audio, it's
   not an instrument, it's a settings screen. You stop exploring.

There is also a hard latency floor underneath both: the sound has to land close enough to the
keypress that your brain fuses them into one event. Miss it and the app is actively unpleasant to
use.

---

## 2. Capturing keystrokes

Global key capture on macOS means `CGEventTap`, gated behind the **Input Monitoring** privacy
permission. The tap is created listen-only:

```swift
CGEvent.tapCreate(
    tap: .cghidEventTap,
    place: .headInsertEventTap,
    options: .listenOnly,   // cannot modify or block input
    eventsOfInterest: CGEventMask(mask),
    callback: callback,
    userInfo: selfPtr
)
```

Klack reads the key code only to distinguish the spacebar. Nothing is logged or stored.

**The thing that costs you an hour if nobody tells you:** `tapCreate` succeeds whether or not
permission has been granted. You get a valid `CFMachPort` back either way. Without permission the
callback simply never fires. There is no error, no exception, no nil — just silence.

This is a genuinely bad failure mode when the app's only output is *sound*, because "no click" is
indistinguishable from "audio is broken," "the app crashed," "the wrong buffer loaded," or "the
volume is zero." Five candidate causes, one symptom.

The fix was a separate 85-line executable, `KlackDiag`, that isolates exactly one variable — no
audio, no menu bar, no app bundle:

```
=== Klack Diagnostic ===
[status] Input Monitoring granted = true
[status] Event tap created successfully!
[status] Type anywhere now — you should see TICK lines appear below:
TICK #1 keyDown keyCode=8
TICK #2 keyDown keyCode=14
```

It converts an auditory, ambiguous symptom into a visible, unambiguous one. It also sidesteps the
bundle-identity problem entirely — run it from Terminal and the permission it needs is Terminal's,
which the user already understands.

A second permission subtlety: an ad-hoc-signed app gets a fresh identity on each rebuild, so macOS
forgets the grant every time you rebuild. That is documented in the README rather than worked
around, because there is no way around it.

There are two smaller correctness details in `KeyTapManager` worth naming. The tap holds a
**non-owning** pointer back to the manager (`passUnretained`), so it must not outlive it — `deinit`
calls `stop()`, otherwise the callback resurrects freed memory. And `stop()` removes the run loop
source from the run loop the tap was actually attached to, not from whichever run loop happens to
be calling `stop()`.

---

## 3. The audio problem: overlapping clicks

Naive playback — one player, retrigger it per keystroke — breaks the instant you actually type.
Clicks have a decay tail of up to 90 ms. Type at 120 WPM and you're issuing a keystroke roughly
every 100 ms, and each one has both a down and an up event. Retriggering one player truncates the
previous tail mid-ring, which is audible and sounds cheap.

The fix is a **round-robin pool of `AVAudioPlayerNode`s** on a single `AVAudioEngine` graph — 24
of them. Since each keystroke consumes two slots, that's about 12 keystrokes of real polyphony:
enough that a fast burst doesn't wrap the ring and cut a still-ringing tail.

The allocator isn't purely round-robin. It scans for an idle node first, and only falls back to
advancing the ring (accepting an audible cut) when every node is genuinely busy:

```swift
private func nextNode() -> AVAudioPlayerNode {
    for offset in 0..<pool.count {
        let candidate = pool[(poolIndex + offset) % pool.count]
        if !candidate.isPlaying {
            poolIndex = (poolIndex + offset + 1) % pool.count
            return candidate
        }
    }
    // everything busy — accept the cut
    let node = pool[poolIndex]
    poolIndex = (poolIndex + 1) % pool.count
    return node
}
```

### The clipping bug, and the wrong fix

Once overlapping playback worked, fast typing sounded *bad* — harsh and crunchy, not just loud.

The tempting fix is to throttle: skip clicks when they arrive too quickly. That's wrong, and it's
worth being precise about why. Throttling changes the app's behaviour to hide a signal-level
problem. You'd be dropping real keystrokes — losing the thing the app exists to do — to avoid a
distortion artifact.

The actual cause is summing. Twelve overlapping clicks, each normalized to 0.85 peak, sum well past
1.0 and clip at the output. The correct fix is a limiter after the mixer:

```swift
let limiterDescription = AudioComponentDescription(
    componentType: kAudioUnitType_Effect,
    componentSubType: kAudioUnitSubType_DynamicsProcessor,
    componentManufacturer: kAudioUnitManufacturer_Apple,
    componentFlags: 0, componentFlagsMask: 0
)
```

Signal-level problems get signal-level fixes. Every click still plays; the summed output is held
under the ceiling. This is one of those bugs where the obvious fix and the correct fix point in
opposite directions.

A related detail: player nodes are connected with a specific `AVAudioFormat` at init. Import a
pack at a different sample rate and scheduling its buffer throws an **uncatchable** NSException —
not a Swift error you can `try`. So the graph rewires itself whenever the format actually changes,
and pack loading explicitly rejects a valid-but-empty buffer, since a failed read otherwise "loads
successfully" and plays silence.

---

## 4. Making a synthesized click not sound synthetic

This was the real work.

### Why filtered noise fails

The obvious approach to a percussive click is a short burst of noise through a bandpass filter,
with a fast decay envelope. It's cheap, it's easy, and it sounds like a pop. It does not sound
like an object being struck.

The reason is physical. When you strike a solid object, the impact excites the object's **resonant
modes** — specific frequencies at which that particular geometry and material prefer to vibrate,
each decaying at its own rate. That mode structure is the object's fingerprint. It's how you can
hear the difference between a tapped wine glass and a tapped coffee mug without looking. Filtered
noise has no mode structure at all. It's energy in a frequency band, and your ear reads it as
"noise that was shaped," not "a thing that was hit."

### Modal synthesis

So Klack models the click as a struck object:

```
click = noise transient          (the mechanical knock of impact)
      + bank of decaying sines   (the resonant modes)
      + low-frequency resonance  (the body — the "thock")
```

The mode ratios are the important part. They are **inharmonic**:

```swift
private static let modeRatios: [Float] = [1.0, 1.03, 1.55, 2.1, 2.42, 2.85]
private static let baseModeDecaysMs: [Float] = [11, 10, 7, 5, 4.5, 3.5]
private static let baseModeAmps: [Float] = [0.22, 0.16, 0.17, 0.12, 0.09, 0.08]
```

Not 1×, 2×, 3×. Integer ratios are what a vibrating string or air column produces, and your ear
has a name for that: a musical note. Plastic and metal parts are irregular geometry, their modes
land at non-integer ratios, and that inharmonicity is precisely what makes something read as *an
object* rather than *a tone*.

Note also the decay times: higher modes decay faster (11 ms down to 3.5 ms). That's not arbitrary
either — high-frequency energy dissipates faster in real materials. Give every mode the same decay
and the result sounds artificially sustained.

### The details that closed the gap

Modal synthesis got it into the right neighbourhood. It still sounded a bit *too clean* — correct,
but manufactured. Five additions closed the gap, each targeting a specific way real audio is messier
than an ideal model:

**Detuned mode pairs.** Note the first two ratios: `1.0` and `1.03`. Two modes ~3% apart beat
against each other, producing a slow amplitude wobble. Real struck materials never ring at one
perfectly pure frequency — there's always a cluster. This is the single change that most reduced
the "synthesized" quality.

**Two-layer noise transient.** A single uniform noise decay reads as a clean synthetic pop.
Layering two bursts with different grain and decay rates (70/30 mix, one decaying 2.4× slower)
gives the impact texture.

**A second delayed impulse.** Real switches have a spring and a keycap; contact isn't one
instantaneous event. A quieter second hit ~1.6 ms later (randomized ±0.3 ms) models the
double-contact:

```swift
let secondHitDelayMs: Float = 1.6 + rng.nextUniform(-0.3, 0.3)
```

**Noise riding on the modes.** Each mode carries a little one-pole-smoothed noise (88% tone /
12% noise) rather than being a pure sine. Raw noise would turn the mode into static; smoothed noise
adds grit. Real resonances aren't infinitely pure.

**Soft-clip saturation.** The whole signal goes through `tanh(x * 1.4)` at the end. Real audio picks
up odd-harmonic distortion passing through actual mechanics and air. A perfectly clean sum of ideal
sine waves does not, and the absence is audible even if you can't name it.

There's also a 0.3 ms attack ramp on the first samples — without it, starting a waveform at nonzero
amplitude produces a digital click at sample 0, an artifact of the *playback*, not the sound.

Every one of these is the same move: identify a specific way the ideal mathematical model is
cleaner than physical reality, and add that imperfection back deliberately.

### Reverb

The `airy` profile needed a spatial tail, so there's a small **Schroeder reverb**: four parallel
comb filters at mutually non-harmonic delays (29.7, 37.1, 41.3, 43.7 ms) producing the diffuse
wash, feeding two allpass stages (5.0 and 1.7 ms) for extra smearing. The comb delays are
deliberately non-integer-related — related delays would reinforce each other into an audible
metallic ringing rather than a smooth tail.

### Two implementations, on purpose

The synthesis exists twice: `Resources/generate_sounds.py` and `Sources/Klack/SoundSynth.swift`.

That's normally a smell, but here it's the correct split. Python is where the eight sound profiles
were designed offline — fast to iterate, no rebuild, and the resulting WAVs ship with the app.
Swift is the live path: **a full 70 ms click renders in 0.165 ms**, which is what makes dragging
a knob feel like an instrument instead of a form field.

The Python file keeps the design notes. The Swift port is the one that had to be fast.

---

## 5. The live mixer

Six parameters, chosen to be things you can *hear* rather than things that are convenient to
implement:

| Parameter | Maps to |
| --- | --- |
| Pitch | Base frequency of the mode bank (400–3200 Hz) |
| Body | Level of the low-frequency resonance |
| Brightness | Level of the upper modes |
| Snappiness | Decay scaling — 1.8× longer at 0, 0.5× shorter at 1 |
| Attack | Noise-transient mix |
| Reverb | Schroeder wet mix |

Two details make it feel live rather than like a preferences pane.

**Every parameter change plays the sound immediately.** Drag a knob, hear the result mid-drag. The
0.165 ms render makes this possible; samples go straight into the player pool with no disk round-trip.

**The mixer lives in the menu bar itself.** Sliders are embedded directly in `NSMenuItem` views —
the same pattern macOS uses for its own volume and brightness sliders. AppKit routes drag events to
the embedded `NSSlider` without closing the menu, so you can dial in a sound without ever opening a
window.

Saving renders the full variant set — 6 down, 3 up, 2 spacebar — with per-variant pitch jitter
(±6% on downs, +10–30% on ups) so rapid typing doesn't sound like a loop, writes them to
`~/Library/Application Support/Klack/CustomProfiles/`, and switches to the new pack. A saved mix is
structurally identical to a shipped pack.

The synthesis uses a seeded xorshift PRNG, so a given seed always renders identically — preview and
save produce the same sound, which would otherwise be a maddening bug.

---

## 6. The import pipeline

Synthesis gives you infinite tweakability. It cannot give you *your* keyboard. So there's a second
path: record your board, get a pack.

`import_recording.py` does onset detection and slicing:

1. **Convert** anything `afconvert` handles to 44.1 kHz mono 16-bit PCM.
2. **Detect onsets** — 5 ms moving-RMS envelope, threshold set 25% of the way from the noise floor
   to the peak, 120 ms debounce.
3. **Slice** 90 ms per onset, with 3 ms pre-roll and an 8 ms fade-out.
4. **Fill** the variant set, resampling the real hits if there aren't enough.

Three choices in there are load-bearing:

**The noise floor is the 10th percentile of the envelope, not the minimum.** The minimum is a
single sample and will happily be a dropout or a digital-silence gap, which sets the threshold
absurdly low and detects every breath in the room as a keystroke. The 10th percentile is robust to
outliers, which is the entire point.

**3 ms of pre-roll before each onset.** An RMS-envelope detector fires when energy has already
*risen* — by definition slightly after the true attack. Slicing at the detected onset decapitates
the transient, which is the sharpest and most identifying part of a click. Pre-roll recovers it.

**8 ms fade-out at the tail.** Cutting a decaying waveform at a nonzero sample produces a
discontinuity, and a discontinuity is a click — an artificial one, layered on top of the real one
you're trying to capture.

If a recording yields fewer distinct hits than variants needed, the extras come from resampling the
real ones (±6%). This is honest about what it is: the underlying material is always the real
recording, and the jitter only prevents literal repetition, which the ear picks up immediately
during fast typing.

---

## 7. The UI, and why Figma came first

The control panel is skeuomorphic — a dark chassis, a glossy white plastic plate, a backlit amber
display, machined rotary knobs. Everything is drawn in Core Graphics. **There are no image assets
in the project.**

### The coding-blind problem

The first attempt went straight to code. Write drawing code, build, run, look, adjust, repeat.

This does not work well, and the reason is worth stating plainly: a build-run-look cycle is a slow,
high-latency feedback loop for a task whose entire success criterion is *visual*. You end up
guessing at values, compiling, discovering the result is wrong in a way you couldn't have predicted
from the source, and guessing again. Worse, working that way you tend to converge on whatever is
*easy to express in Core Graphics* rather than what actually looks right — the tool quietly starts
choosing the design.

The fix was to **design it in Figma first**. Not as a formality — specifically so the visual target
existed, and could be *seen and judged*, before a line of drawing code was written. Then the code
becomes a port with a reference to check against, not an exploration in a medium with a 30-second
feedback loop.

Two concrete things fell out of that:

**Every custom view sets `isFlipped = true`.** Figma uses a top-left origin; AppKit defaults to
bottom-left. Flipping the views means coordinates transfer from the reference 1:1, with no mental
arithmetic per value. That single decision removed an entire category of transcription error.

**The Figma draft was wrong in a way that was only visible once built,** and that's fine — that's
what a reference is for. The chassis in the draft was too light; against it the white plate read
washed out. The shipped chassis runs meaningfully darker. Having a reference doesn't mean following
it off a cliff; it means having something specific to disagree with.

### V1: brushed metal

The first skin (`MetalStyleViews.swift`) was a brushed-metal faceplate with vertical faders, a
green LCD, and corner screws.

Two techniques from it are worth keeping:

**Brushed metal without noise.** Three striation layers at periods 3, 5, and 7 pixels. Since those
share no common factor, the composite pattern only repeats at their LCM — 105 pixels — which reads
as random brushing while being fully deterministic and cheap.

**Inner shadow, which AppKit has no primitive for.** The standard trick: clip to the shape, then
fill it with `.sourceOut` blend mode inside nested transparency layers, so the shadow of the
*inverted* shape lands inside the path. Both nested layers are required — the outer applies alpha
uniformly, the inner gives `.sourceOut` a defined backdrop. This is what makes the fader tracks and
the LCD read as machined *into* the faceplate rather than pasted on.

The LCD used **DSEG7 Classic** (SIL OFL) for genuine seven-segment glyphs, with three authenticity
details: ghost segments (drawing `888` faintly behind the value, since DSEG renders 8 as all
segments lit — that's what unlit segments on a real LCD look like), a scanline and RGB subpixel
mask, and chromatic aberration (the same text offset half a pixel in red and blue under the main
glyphs, mimicking phosphor colour fringing).

### V2: glossy plastic

V1 was competent but generic — it looked like every skeuomorphic audio plugin from 2008. V2
(`PlasticSkinViews.swift`) went for injection-moulded plastic and a backlit amber display instead.

The observations that made it work:

**White plastic is never neutral grey.** The body runs cool at the highlight and warm in shadow.
Sampling a flat grey for a "white plastic" surface is the single fastest way to make it look like a
rectangle instead of a moulded part.

**Gloss is a hard terminator, not a soft airbrush.** The gradient has a near-zero-width step at its
midpoint:

```swift
(0.5000, NSColor(white: 1, alpha: 0.30)),
(0.5001, NSColor(white: 1, alpha: 0.0)),
```

That abrupt edge is what reads as a lacquered surface. Smoothly fading it produces something that
looks fogged.

**A screen has to look like it's emitting, not filled.** Four cues, and it needs all four: an
off-centre bloom (light source behind and slightly above), a vignette *tighter* than the bloom (the
radius mismatch creates depth behind the glass), a light leak at the bezel (stroked at widening
radii with falling alpha, since Core Graphics can't feather a stroke), and faint grain. Plus a bloom
shadow spilling onto the housing — without it the display still looks pasted on.

**Grain stays under 5% alpha.** Above that it stops reading as surface texture and starts reading
as dirt.

The display isn't decorative: it shows the live pack name, volume, real status flags, and a
waveform driven by actual keystrokes, and its backlight lifts on each keypress and decays at 30 Hz.

---

## 8. The debugging story: seeing your own work

Here's the problem with an AI writing UI drawing code. The code compiles. It's well-formed. It's
even well-reasoned. And it can be *completely wrong* in a way that's invisible in the source and
obvious in one glance at the output.

The response was a **headless render harness**:

```bash
KLACK_RENDER_TEST=/tmp/panel.png .build/release/Klack
```

It builds the control panel offscreen, populates the waveform so the trace isn't flat, draws the
content view via `cacheDisplay(in:to:)`, writes a PNG, and exits. Now the drawing code can be
checked by *looking at the result*, not by reading the source and hoping.

It caught three real bugs that code review had not.

### Bug 1 — the knob rendered upside-down

The knob body is expensive (hundreds of stroked spokes), so it's rendered once into a cached image
and only the indicator and value arc redraw per frame. Sensible optimization.

It rendered vertically flipped. The highlight sat at the bottom, the shadow at the top — a knob lit
from below, which is exactly the kind of thing your visual system flags instantly and your
code-reading brain does not. The cause: the views are all `isFlipped = true`, and drawing a raw
`CGImage` into a flipped context silently flips it.

The fix is to render through `NSImage(size:flipped:)` so the cached body is drawn in the same
coordinate space as the view:

```swift
let image = NSImage(size: size, flipped: true) { [weak self] _ in
    self?.drawBodyContents()
    return true
}
```

Nothing in the source looks wrong. The bug lives in an interaction between a caching decision and a
coordinate-system decision made in two different places.

### Bug 2 — the starburst

The knob's lathe finish is drawn as hundreds of fine radial spokes. The first version had them run
from the centre outward, which is the obvious thing to do.

The render showed a bright starburst at the middle of every knob.

The reason is geometric and, in hindsight, unavoidable: **radial lines converge at the centre.**
Near the middle, hundreds of strokes occupy a few pixels, so ink density — and therefore
brightness — spikes. Each individual stroke is at 1–3% alpha and looks entirely reasonable in the
source. The artifact is a property of the *arrangement*, not any one line.

Two fixes together: start the spokes well out from the middle (34–44% of the radius), and jitter
each one's inner and outer extent so they don't share a hard edge:

```swift
p.move(to: point(a, r * (0.34 + 0.10 * jitter), c))
p.line(to: point(a, r * (0.94 + 0.05 * jitter), c))
```

### Bug 3 — the inverted drawer

The mixer drawer's contents were laid out upside-down. Cause: a plain `NSView` used as a container
lays out its children bottom-up, while its own frame is positioned top-down by the surrounding
flipped code. The container silently inverts everything inside it.

Fix: a two-line `FlippedContainer` type, used for every container in the skin. The value is in
making the convention explicit and uniform rather than remembering it case by case.

### What the harness actually proves

All three bugs share a shape: **the source is locally correct, and the composition is wrong.** Every
individual decision is defensible. The bug lives in the interaction between decisions made in
different places — and that interaction is only visible in the rendered pixels.

You cannot review your way to these. You have to look. The harness is what made looking possible for
an agent that has no eyes on the screen, and it's the piece of this project I'd port to any future
UI work first.

---

## 9. Research beat intuition — twice

Two things were confidently implemented wrong, and corrected only by going and checking.

**Radial brushing is anisotropic in *angle*, not radius.** The first lathe-finish implementation
drew concentric rings of varying brightness. That's the intuitive reading of "the brightness varies
across a turned surface" — but it's backwards. On a lathe-turned part, the tool marks run
circumferentially, and the specular response varies with the **angle** you view a point from, not
its distance from the centre. Concentric rings look like a target. Fine radial spokes with a
per-angle hashed alpha look like machined metal. The corrected version also carries a two-lobe
angular sheen (`cos(2θ)`), which is the actual specular signature of turned metal — and it's kept
deliberately faint, since it should suggest a light direction, not draw a fan across the face.

**Rotary controls sweep ~270°, not 120°.** The first knob used a shallow arc. Real hardware
potentiometers sweep roughly 270–288°, and the near-universal UI convention is 270° centred on 12
o'clock — from −135° to +135°. A 120° sweep looks wrong in a way that's hard to articulate but
immediately legible: the knob appears barely turnable, and the indicator never points anywhere
convincing.

Both cases follow the same pattern: an intuition that felt obviously right, produced code that
compiled and ran, and was wrong. The tell in both was that the *output* looked subtly off while the
*reasoning* seemed sound — which is the exact signature of a wrong premise rather than a bug.

While fixing the knob, one more decision proved worth it: every angle in `KnobControl` is measured
clockwise from 12 o'clock, and **exactly one function** converts an angle to a point. Arcs are drawn
by sampling points rather than `addArc`, so nothing depends on flipped-context winding conventions.
Sign errors in arc code are endemic; one convention and one conversion point eliminates the class.

---

## 10. Small things that mattered

**Typing-speed reward.** Type faster and the volume rises up to +30%, computed over a 6-keystroke
rolling window and ramping in between 180 ms and 80 ms average inter-key intervals. It's a tiny
piece of positive feedback for flow, and it also drives the panel: the waveform trace climbs, the
FAST BOOST flag lights, and the display backlight lifts.

**Vertical drag on knobs.** Not rotational. Tracing a circle with a mouse is imprecise and
frustrating; every serious audio UI uses vertical drag for exactly this reason. Option-drag gives
quarter-speed fine adjustment.

**Battery.** The permission poller runs every 3 seconds at startup and **cancels itself** the moment
the tap goes live. This is a menu-bar app that runs all day; a permanent 3-second wakeup for a
condition that can only change once is pure battery cost for nothing.

**Failure handling in pack switching.** If switching packs fails, it reverts to the previous pack —
and if the revert *also* fails, it logs explicitly that audio may be silent, rather than leaving a
mystery.

---

## 11. Licensing

Worth stating because audio projects tend to be sloppy here.

Every sound in Klack is either **procedurally synthesized** (`generate_sounds.py` /
`SoundSynth.swift` — no sampled material at all) or **imported from properly licensed sources**. No
scraped sample libraries.

The **DSEG7 Classic** font is **SIL Open Font License 1.1**, and is bundled together with its full
license text at `Resources/fonts/DSEG-OFL-LICENSE.txt`. Fonts are registered at runtime via
`CTFontManagerRegisterFontsForURL` with a monospace fallback if registration fails.

Zero third-party code dependencies. Everything else is system frameworks.

---

## 12. What I'd take to the next project

**Design where you can see it, then port.** The Figma-first decision was the highest-leverage change
in the whole build. A visual problem needs a visual feedback loop; a compile-run-look cycle is too
slow and too lossy, and it quietly biases the design toward whatever the drawing API makes easy.

**Build the harness that lets you check your own work.** The render harness took very little code
and caught three bugs that were invisible in code review — because all three were composition
errors, not logic errors. The general principle: if the output is a medium you can't directly
perceive, build the thing that renders it into one you can. That applies well beyond graphics.

**Isolate one variable when a symptom is ambiguous.** `KlackDiag` exists because "no sound" had five
plausible causes. 85 lines that eliminate four of them are worth more than an hour of reasoning
about which is likeliest.

**Fix problems at the level they occur.** Throttling keystrokes to fix clipping would have degraded
the core feature to hide a signal-level artifact. A limiter fixes it where it happens. The
attractive fix and the correct fix pointed in opposite directions.

**When the model sounds clean and reality doesn't, add the mess back deliberately.** Detuned mode
pairs, layered noise transients, the double-contact impulse, noise riding on the modes, soft-clip
saturation — every one is a specific, named way physical audio is dirtier than ideal math. That's
the whole difference between a click that sounds like an object and one that sounds like a computer.

**Check the premise when the reasoning looks sound and the output doesn't.** Both the anisotropy
mistake and the sweep-angle mistake produced code that was internally consistent and externally
wrong. That combination is the signature of a bad assumption, and it's cheaper to go verify than to
keep debugging downstream of it.
