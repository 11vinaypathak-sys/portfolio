# Klack

A macOS menu-bar app that plays mechanical-keyboard sounds on every keystroke.

Klack sits in the menu bar, listens to global key events, and fires a short click sample on
each key down and key up. It ships with a set of recorded and procedurally-synthesized sound
packs, a live synthesis mixer for designing your own click, and an import pipeline for turning
a recording of your actual keyboard into a playable pack.



**The reel.** The 84-second film about how Klack was made — the story, the process, the first
click — is on Instagram: **[instagram.com/p/Dc5rn_OzGjC](https://www.instagram.com/p/Dc5rn_OzGjC/)**.
The full-resolution MP4 is attached to the

![Klack control panel](docs/images/demo.gif)

*The panel responding to live typing — the waveform, backlight, and status flags are driven by real keystrokes. ([higher-quality MP4](docs/images/demo.mp4))*

![Klack control panel](docs/images/panel.png)

The mixer drawer, opened from the MIXER tab:

![Klack with mixer drawer open](docs/images/panel-mixer.png)

*Both images are produced from source by the render harness — see [Render harness](#render-harness).*

---

## Download


1. Open the DMG and drag **Klack** into **Applications**.
2. First launch: **right-click Klack.app → Open → Open.** Klack is signed but not notarized
   (no Apple Developer account behind it), so a plain double-click shows "cannot be opened".
   Right-click → Open is Apple's sanctioned way past that, and you only do it once. If macOS
   still refuses, run `xattr -dr com.apple.quarantine /Applications/Klack.app` in Terminal.
3. A system dialog asks to allow Klack to **monitor input**. Allow it, flip Klack **on** in
   System Settings → Privacy & Security → Input Monitoring, and start typing. No relaunch
   needed — the app notices within three seconds.

That is the whole install. If step 3 never shows a dialog, see
[Input Monitoring permission](#input-monitoring-permission).

**Other platforms.** A lean Windows tray-app port lives under [`windows/`](windows/) and an
Android keyboard (phones don't let an app hear other apps' keys, so there it *is* the keyboard)
under [`android/`](android/). Both are works in progress and their READMEs say exactly what has
and hasn't been tested.

---

## What it does

- **Global keystroke sounds.** Every key down and key up plays a sample. Space gets its own
  deeper down/up pair, because on a real board it does.
- **Sound packs.** Switch between packs from the menu bar or the control panel. Packs are just
  folders of WAV files, so adding one is a filesystem operation.
- **Live synth mixer.** Six parameters — pitch, body, brightness, snappiness, attack, reverb —
  re-render the click in real time as you drag. Save a mix as a named custom pack.
- **Import your own keyboard.** Record yourself typing, run one script, get a pack built from
  your real switches.
- **Typing-speed reward.** Type faster and the volume rises slightly (up to +30%), then settles
  back down when you slow.
- **Live waveform.** The control panel display shows a scrolling trace driven by actual
  keystrokes, not an animation loop.

The everyday interface is the menu-bar dropdown. The control panel is the full version, with
knobs.

---

## Requirements

- macOS 12 or later
- Swift 5.9+ toolchain (Xcode or Command Line Tools) — only to build from source; the DMG needs nothing
- Python 3 — only if you want to regenerate or import sounds

---

## Build from source

Klack is a SwiftPM package with no external dependencies. One script builds the binary,
assembles the bundle, generates the synthesized sound packs on a fresh clone, signs it with a
stable identity, runs three self-test suites, and relaunches the app:

```bash
cd klack
./build-scripts/install.sh
```

A keyboard glyph appears in the menu bar. There is no Dock icon and no main window —
`LSUIElement` is `true` in `Info.plist`.

Things the script does that are easy to get wrong by hand, and why they matter:

- **Signs with a pinned designated requirement.** An ad-hoc signature normally derives its
  identity from the binary's hash, so every rebuild is a new app to macOS and the Input
  Monitoring grant silently dies. The script pins the requirement to the bundle identifier and
  then verifies it took, failing the build if it didn't.
- **Refuses to ship broken audio.** Audio health across every pack, burst shaping, and crash
  survival on audio-device change all run before launch. A failure exits non-zero.
- **Quits the running copy first.** Replacing a binary under a live process leaves the old
  code in memory and makes a fresh build look like it changed nothing.

After a code change, run the same script again.

---

## Input Monitoring permission

Klack reads global key events through a `CGEventTap`. macOS gates that behind the **Input
Monitoring** privacy permission.

The tap is created in listen-only mode — Klack never modifies, blocks, or forwards your
keystrokes, and never records what you type. It reads the key code only to tell the spacebar
apart from everything else.

**To grant it:**

1. Open Klack's menu-bar dropdown.
2. Click **Grant Input Monitoring Access…**. This triggers the system prompt and opens
   System Settings → Privacy & Security → Input Monitoring.
3. Enable **Klack** in the list.

Klack retries the tap every 3 seconds until it is live, then cancels the timer so it isn't
waking the CPU all day. It gates on whether the tap actually exists, never on a permission
flag — on macOS 26 the flags and the capability disagree in both directions. If the tap is
missing, the **Grant Input Monitoring Access…** item stays visible in the menu.

### Toggle is on but it still hears nothing

macOS is holding a record for an older build of Klack and matching the new one against it.
Nothing you do in System Settings fixes that; clear the record instead:

```bash
tccutil reset ListenEvent com.vinaypathak.klack
```

The permission dialog appears again within a few seconds. Allow it and you're done.

### If it still doesn't work

There is a standalone diagnostic that isolates the permission mechanism from everything else —
no audio, no menu bar, no bundle:

```bash
swift build -c release --product KlackDiag
.build/release/KlackDiag
```

Run it in your own Terminal and type. It prints the permission status, whether `tapCreate`
succeeded, and a `TICK` line per keystroke. If you see TICKs, the tap works and the problem is
elsewhere. If you don't, it's the permission — and the permission it needs is Terminal's, which
is much easier to reason about than an app bundle's.

---

## Using it

### Menu bar

| Item | What it does |
| --- | --- |
| Open Control Panel Window… | Opens the full knob panel |
| Enabled | Master on/off for the key tap |
| *(Sound Pack list)* | Select the active pack; selecting one plays a preview |
| Play Test Sound (⌘T) | Fires one click |
| Volume | Master output volume |
| Design Your Own | Six live synth sliders (below) |
| Preview My Mix (⌘P) | Renders and plays the current mix |
| Save as My Sound | Writes the mix out as a pack and switches to it |
| Launch at Login | macOS 13+, via `SMAppService` |
| Quit Klack (⌘Q) | |

Dragging a mixer slider re-renders and plays the click immediately — the menu stays open while
you drag, so you can dial a sound in without reopening anything.

### Control panel

Open it from the menu. It's a borderless window; drag it anywhere by its body.

- **Amber display** — active pack, volume, live status flags (ENABLED / FAST BOOST / SPACEBAR),
  and the live keystroke waveform. The backlight lifts slightly on each keystroke and decays.
- **Lamps down the left edge** — pack selection, one per pack.
- **TONE / BODY knobs** — the two synth parameters you reach for most.
- **Large knob** — master volume.
- **▶ and SAVE buttons** — preview the current mix, save it as a pack.
- **MIXER tab on the right edge** — slides out a drawer with the remaining four parameters
  (BRIGHT, SNAP, ATTACK, REVERB) and a Reset button.

All knobs use **vertical drag**, the convention audio hardware UIs use because tracing a circle
with a mouse is imprecise. Hold **Option** while dragging for fine adjustment (quarter speed).

### The six mixer parameters

| Parameter | Range | What it changes |
| --- | --- | --- |
| Pitch | 400–3200 Hz | Base frequency of the resonant mode bank |
| Body / Thock | 0–1 | Level of the low-frequency body resonance |
| Brightness / Click | 0–1 | Level of the upper resonant modes |
| Snappiness | 0–1 | Decay time scaling — 0 is loose and long, 1 is tight and short |
| Attack | 0–1 | Noise-transient mix; the mechanical bite of impact |
| Reverb | 0–1 | Wet mix of a small Schroeder reverb |

---

## Importing your own keyboard

`Resources/import_recording.py` turns raw recordings into a Klack pack.

```bash
python3 Resources/import_recording.py --name my_keyboard recording.m4a
```

You can pass several files at once:

```bash
python3 Resources/import_recording.py --name my_keyboard take1.wav take2.wav take3.m4a
```

**How to record.** Get your phone or a mic close to the board, in a quiet room, and type maybe
ten to twenty deliberate single keystrokes with clear gaps between them. Gaps matter — the
importer separates keystrokes by silence.

**What the script does.**

1. Converts any format `afconvert` handles (wav/mp3/m4a/aac/caf/…) to 44.1 kHz mono 16-bit PCM.
2. Detects keystroke onsets with an energy-based detector: a 5 ms moving-RMS envelope,
   thresholded at 25% of the way from the noise floor to the peak, with a 120 ms debounce so one
   keystroke doesn't register twice.
3. Slices a 90 ms window per onset, with 3 ms of pre-roll so the attack isn't clipped and an
   8 ms fade-out so the tail doesn't click.
4. Writes the full variant set Klack expects: `click_down_1..6.wav`, `click_up_1..3.wav`,
   `click_space_down.wav`, `click_space_up.wav`.

If it finds fewer distinct hits than variants needed, it fills the rest by lightly resampling
the real ones (±6% pitch/speed) so fast typing doesn't sound like a loop. Key-up variants are
pitched up 8–20% and dropped to 60% gain; the spacebar is pitched down to 85%. The underlying
material is always your recording — the jitter only prevents literal repeats.

Output goes to `~/Library/Application Support/Klack/CustomProfiles/<name>/`. Restart Klack (or
save a mix, which rebuilds the menu) and the pack appears under **Sound Pack**.

Use `--out-root` to write somewhere else.

### Regenerating the synthesized packs

`Resources/generate_sounds.py` regenerates the procedural packs into `Resources/profiles/`:

```bash
python3 Resources/generate_sounds.py
```

It defines eight profiles — `the_bench`, `deep_thock`, `creamy`, `clacky`, `poppy`, `clicky`,
`vintage`, and `airy` — each a distinct set of modal-synthesis parameters. Editing the `PROFILES`
dictionary and re-running is how you add a new one.

Note that the built-in profile lists in `AudioEngine.swift`
(`signatureProfileNames`, `extraProfileNames`, `builtInProfileNames`) are currently empty, so
only imported/custom packs show in the menu. Repopulate them to expose the synthesized packs.

### Pack format

A pack is a directory containing:

```
click_down_1.wav ... click_down_6.wav   # key down variants, picked at random
click_up_1.wav   ... click_up_3.wav     # key up variants
click_space_down.wav
click_space_up.wav
```

Only `click_down_1.wav` is strictly required for the directory to be recognized as a pack. All
files in a pack must share a sample rate and channel count; the audio graph rewires itself when
you switch to a pack with a different format.

---

## How it works

**Key capture** (`KeyTapManager.swift`) — a `CGEventTap` on `.cghidEventTap` in `.listenOnly`
mode, masked to `keyDown` and `keyUp`, attached to the main run loop. Note that `tapCreate`
succeeds whether or not you have permission; without it the callback simply never fires. That
asymmetry is why the diagnostic tool exists.

**Playback** (`AudioEngine.swift`) — one `AVAudioEngine` graph with a round-robin pool of 24
`AVAudioPlayerNode`s. Each keystroke consumes two slots (down + up), so that's roughly 12
keystrokes of polyphony. The allocator prefers an idle node so a still-ringing 90 ms tail is
never truncated, and only wraps the ring when everything is genuinely busy. A
`kAudioUnitSubType_DynamicsProcessor` limiter sits between the mixer and the output — fast typing
sums many overlapping clicks, and without the limiter that clips into a harsh mess.

**Synthesis** (`SoundSynth.swift`) — modal synthesis. A noise transient plus six decaying sine
modes at inharmonic ratios plus a low-frequency body resonance, with detuned mode pairs for
natural beating, a second delayed impulse for switch double-contact, a Schroeder reverb
(4 parallel combs → 2 allpass stages), and `tanh` soft-clip saturation. A full 70 ms click
renders in 0.165 ms (measured, `KLACK_BENCH=1`), which is what makes the mixer feel live. The Python original in
`Resources/generate_sounds.py` carries the design notes; the Swift port exists so the mixer
doesn't need a Python round-trip or an app rebuild.

**UI** (`PlasticSkinViews.swift`, `MetalStyleViews.swift`) — every surface is drawn in Core
Graphics. No image assets. `MetalStyleViews.swift` is the earlier brushed-metal skin;
`PlasticSkinViews.swift` is the current glossy-plastic one.

See [CASE_STUDY.md](CASE_STUDY.md) for the reasoning behind these choices.

---

## Development

### Headless render harness

The control panel can render itself offscreen to a PNG without a human looking at a screen:

```bash
KLACK_RENDER_TEST=/tmp/panel.png .build/release/Klack
KLACK_RENDER_TEST=/tmp/panel.png KLACK_RENDER_MIXER=1 .build/release/Klack
```

It builds the window, populates the waveform with fake activity so the trace isn't flat,
optionally opens the mixer drawer, draws the content view via `cacheDisplay(in:to:)`, writes the
PNG, and exits. Setting `KLACK_RENDER_MIXER` also opens the drawer without the window animation.

This was not a nice-to-have. See the case study for the three rendering bugs it caught that code
review did not.

### Layout convention

Every custom view in the plastic skin sets `isFlipped = true`, so coordinates match the Figma
reference's top-left origin directly. `FlippedContainer` exists because a plain `NSView` parent
lays out children bottom-up while its own frame is positioned top-down, which silently inverts
its contents.

Every angle in `KnobControl` is measured clockwise from 12 o'clock, converted to a point in
exactly one place. Arcs are drawn by sampling points rather than `addArc`, so the result doesn't
depend on flipped-context winding conventions.

---

## Privacy

Klack reads global key events to know when to play a sound. It uses only the key code, and only
to check whether the key was the spacebar. It does not log, store, transmit, or interpret
keystrokes. The event tap is listen-only and cannot modify or block input. There is no network
code in the app.

---

## Credits & License

**DSEG7 Classic** — the seven-segment display font used in the metal skin's LCD readout, by
Keshikan. Licensed under the **SIL Open Font License 1.1**. The font and its full license text
are bundled at `Resources/fonts/DSEG7Classic-Bold.ttf` and
`Resources/fonts/DSEG-OFL-LICENSE.txt`. Fonts are registered at runtime via
`CTFontManagerRegisterFontsForURL` with a monospace fallback.

**Audio** — every sound in this project is either synthesized procedurally by
`generate_sounds.py` / `SoundSynth.swift` (no sampled material) or imported from properly
licensed recordings. No scraped or unlicensed sample libraries are used.

**Dependencies** — none. AVFoundation, AudioToolbox, CoreGraphics, Cocoa, and ServiceManagement
only, all from the system.

Built by Vinay Pathak, with Claude.
