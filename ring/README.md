# Shift

Two things you're grateful for, one good thing that happened at work. Once a day,
at the end of your shift.

This is **Three Good Things** (Seligman, Steen, Park & Peterson, *American
Psychologist* 2005) — an exercise Vinay agreed with his psychologist, not a tool
idea. That matters for every decision below: the job is to make the exercise easy
to do and impossible to lose, not to be clever about it.

Named for the shift rather than the count, so the name survives the exercise
changing shape. It is also deliberately a word that gives nothing away: this
record may go to a psychologist, and the app's name appears in the Dock, in
Control Center and in every screen share. "Three Good Things" would announce the
contents; "Shift" does not.

It was called **sh–1** until the Control Center work, on a Teenage Engineering
model-code joke. The joke did not survive being printed in a list next to real
app names — it read as a typo. The bundle id and the `sh1` command keep the old
spelling, because nobody reads those and renaming them would orphan every stored
preference.

## Making it start at login

**The simple way, and the one to use:**

> System Settings → General → Login Items & Extensions → **Open at Login** → **+**
> → pick `Shift.app`

Two clicks, nothing to debug. `install.sh` below does the same job with a
LaunchAgent and is entirely optional — it exists because `SMAppService` failed
here and I wanted something programmatic, which was over-engineering for a
checkbox that already exists.

## Optional: install it as a utility

```sh
./install.sh      # and ./uninstall.sh to reverse it
```

That makes it a thing that is simply there rather than an app you remember to
open:

- the bundle goes to `~/Applications/Shift.app`, not a build directory
- a **LaunchAgent** starts it at login and brings it back if it crashes —
  `KeepAlive` is `SuccessfulExit: false`, so choosing Quit actually quits
  instead of respawning in your face
- `sh1` is symlinked into `~/.local/bin`, which is already on PATH
- stderr goes to `~/Library/Logs/sh1.log`

The installer runs `--selftest` and **refuses to install a build whose own
assertions fail.**

**Deliberately a LaunchAgent, not `SMAppService`.** Registration via
`SMAppService` has never been observed succeeding on this machine, and since
notifications failed here for code-signing reasons it may never. launchd does
not care about signing identities. When the plist exists the app notices, drops
its own "open at login" toggle, and shows **Installed — runs at login** instead,
so there is only ever one mechanism in play and it tells the truth.

Nothing needs sudo. Nothing is written outside `$HOME`. `uninstall.sh` removes
the agent and the symlink and **leaves your entries and the app bundle alone** —
uninstalling a tool should never be how you lose a record.

Only one instance ever runs: a second launch asks the first to show its window
and exits.

## Build without installing

```sh
./build.sh
open "build/Shift.app"
./build/Shift.app/Contents/MacOS/sh1 --selftest
```

Command Line Tools only — no Xcode, no `.xcodeproj`.

## Opening it

**There is no menu-bar item.** It was the original way in and it never worked
here: the display is 1728 wide with a 185pt notch, fourteen other items already
hold roughly 630pt of what is left, and macOS culls from that cluster whenever a
frontmost app's menus want the room. Two rounds of work went into making it
survive before it was retired instead.

Two things replaced it. **Ring** is the everyday one — see below. Control Center
is the other:

> Control Center (menu-bar icon, or **fn-C**) → **Edit Controls** → the
> **Shortcuts** category → **Open App** → **Shift**

That needs no Xcode and no signing identity, because it is Apple's own control
opening an ordinary app. A *real* `ControlWidget` — one that shows 1 of 3 on the
tile itself rather than just being a button — would need a WidgetKit extension
target, which means an Xcode project. Control Center has been open to third
parties since macOS Tahoe 26 and a free Personal Team is enough to sign one, so
that door is shut by this repo's toolchain rather than by Apple.

Because the button only opens the app, **the window is the whole interface.** It
carries two things the menu bar used to:

- **Past entries…** opens the record. Without it the dashboard would be
  reachable only from the item that started this problem.
- **Options ▾** is the old status-item menu — schedule, desktop tile, exports,
  Quit. Right-clicking anywhere outside a text field gives the same menu.

Quit also answers to ⌘Q. An accessory app draws no menu bar, but its key
equivalents still resolve against `NSApp.mainMenu`, so there is one installed
purely for the keystrokes. That also fixes something that had been broken since
the beginning: with no Edit menu behind the window, **⌘V did nothing in the
fields**, because Cut/Copy/Paste in Cocoa are menu commands rather than
something `NSTextField` implements.

## Ring — the second program in this repo

```sh
./build-ring.sh     # build/Ring.app
./install-ring.sh   # ~/Applications/Ring.app + a LaunchAgent
```

A 34pt disc that clings to the left or right edge of the screen. Click it and a
short arc of app slots fans out; click a slot to launch; click anywhere else and
it retracts. Shift is one of the slots.

**It is a separate program, and that is not an accident.** Shift is the Three
Good Things record and those entries may end up in front of a psychologist. A
general-purpose app launcher does not belong inside that: different lifetime,
different risk, different reason to exist. Ring is its own process, its own
bundle (`com.vinaypathak.ring`), its own LaunchAgent, and Shift knows nothing
about it.

**It lives in this repo anyway**, because it draws `Aero.plate` — the app icon's
own routine. The commit before Ring was spent fixing exactly the drift two copies
of a visual language produce. If Ring ever earns a life of its own, `Aero.swift`
becomes a shared module and it moves out.

**28 Oct 2026 — liquid glass, Dock motion, one icon per surface.** The nub and
the slots are now `GlassDisc` (ring/Glass.swift): a real `NSVisualEffectView`
blur of whatever is behind the disc, with a dome tint, a specular rim, a
refraction band and a travelling glint on one shared clock. The ring magnifies
like the macOS Dock — the slot under the pointer swells to 1.55×, neighbours
swell by a gaussian of arc distance and slide apart to make room — driven by a
60 Hz timer that runs only while the ring is open. Every ZOLA surface is a link
with its own drawn glyph (`Glyph`): eye, door, market line, ringed planet,
sliders, gauge, archway, mandala; links always show, and a removed one is
remembered as hidden (menu: "Show the ZOLA links again"). Ring is single-
instance: a second copy quits, unless it is a `--open` debug run, which takes
over. Debug: `--open --probe=N --debug` fans the ring out with a fake pointer
parked on slot N, so a `screencapture` shows the swell.

Three decisions worth keeping:

- **Only the horizontal is magnetic.** Let go anywhere and the nub takes the
  nearer vertical edge, keeping its height. Snapping vertically too would leave
  a handful of legal positions instead of an edge to slide along, and the height
  is the part worth choosing — clear of the notch, clear of the Dock. It clings
  3pt off the bezel, not flush, or the shadow clips and it reads as damaged.
- **The arc opens toward the middle of the screen the nub is on.** One rule, no
  per-edge special cases, and it stays on screen in a corner.
- **The ring grows rather than crowds.** With a fixed radius and a capped sweep,
  the seventh app is the one that makes every slot overlap its neighbours —
  silently, all at once, and exactly when the thing has become useful enough to
  fill up. The sweep caps and the radius opens outward to hold the spacing.

### What can go in it

Any `.app`, chosen through the **+** slot or *Add an app…* on the nub's
right-click menu. Right-click a slot to remove it. The list lives in
`UserDefaults` under Shift's suite, key `ringApps`.

Plus **links**, for things that have no bundle to point at. Zola is a daemon on
`127.0.0.1:3737`, and "the thing I open ten times a day" is a better definition
of what belongs in a launcher than "the thing that happens to be an `.app`". A
link has no icon, so it wears a letter — no favicon fetching, because a launcher
has no business making network requests.

**There is deliberately no "add a URL…" box.** The only two ways a target enters
the ring are the Finder picker and a fixed table in `ring/Apps.swift`, so nothing
in the list arrived as an arbitrary string. Adding a link is a one-line source
change, on purpose.

### The desktop tile stood down

Shift has a 56pt tile that can sit on the desktop. It used to be **on** by
default, because it replaced the menu-bar item macOS kept culling. Ring took that
job, and a loose tile in the corner *as well* is two permanent objects doing one
thing — so the default is now off. The code is unchanged and it is still one
click away under **Desktop widget** in the menu.

The tile is drawn by `Aero.plate` at the icon's own proportions, so it is not a
lookalike of the icon; it is the icon. `--render-widget out.png --sample` draws
it with a half-written day, because the lit gems are the state worth judging and
today's real entry is usually empty — and a render harness does not get to write
into the record to make a screenshot look better.

## Using it

Press the Control Center button, or re-open the app.

| | |
|---|---|
| `⇥` | next field, then Later, then Save, then round again |
| `space` | press the focused button |
| `↩` | save and close, from anywhere |
| `esc` / click away / **later** | close without saving |

Re-opening a day you've already filled loads what you wrote and lands you on the
first empty field. Changing it is fine.

Headless:

```sh
sh1 --entry "grateful one" "grateful two" "good at work"
sh1 --who5 4 3 4 2 3      # five values, 5 down to 0
sh1 --status              # today, the week, the schedule, the last score
sh1 --schedule            # what it has learned and why
sh1 --export              # the whole record, plain text
sh1 --login [on|off]      # read back from the system, not echoed
sh1 --nudge               # rehearse the arrival
sh1 --selftest            # assertions; exits non-zero on failure
sh1 --render out.png      # draw a window offscreen
sh1 --render-who5 out.png
sh1 --render-banner out.png
sh1 --render-widget out.png [--sample]
sh1 --widget              # toggle the desktop tile
```

`--render` exists because every pixel is hand-drawn and a live screenshot needs
an unlocked screen and a person at the keyboard. It writes nothing to the log.

`--login` reports what `SMAppService` actually says afterwards rather than
echoing what was asked for, so a failed registration cannot masquerade as
success — the notification spike taught that lesson the hard way.

## AERO — the visual language

Windows 7, rebuilt from measured values rather than remembered ones. It replaced
FLINT — Teenage Engineering: flat, grey, zero radius, no light source — which he
found cold. `Aero.swift` is the whole language; nothing outside it picks a
colour.

Three things make Aero read as rich rather than merely blue, and losing any one
of them loses the look:

1. **The glass is real.** `NSVisualEffectView` with `.behindWindow` blurs and
   saturates what is actually behind the window, which is what the DWM did. A
   painted gradient pretending to be glass is the single most obvious tell.
2. **The gradient steps.** A Win7 button is not a smooth fade — it is a hard
   discontinuity at 45%. Smooth it and you get Windows 8.
3. **The specular.** Diagonal stripes at 54° plus a sheen along the top. Without
   it glass looks like fog.

Sourced, not recalled: the glass composition is the DWM's own shader parameters
(`BlurDeviation` 30, colorization `0x6B74B8FC` "Sky" at 42% alpha); the control
colours are `7.css`'s variables — surface `#F0F0F0`, field border `#ABADB3`,
instruction `#003399`, progress green `#0BD82C`. Where a value is sampled or
inferred rather than cited, the comment in code says so.

Type is Segoe UI 9pt, which is 12px at 96 DPI and the size nearly everything in a
Win7 dialog is set at. Segoe is not licensed for this Mac and Selawik —
Microsoft's own metric-compatible clone — is not installed, so the stack tries
both first and falls back to Inter. `brew install --cask font-selawik` upgrades
the app with no code change.

**The shape grammar survived the migration**, because it is the one thing that
tells the three slots apart: triangle, square, circle — confirmed verbatim by
Jesper Kouthoofd in an SFMOMA interview, and adopted after a version that drew
three identical squares and relied on position alone. What changed is the
material. FLINT drew them as flat fills because FLINT denied a light source;
`Aero.gem` draws them as gel — the stepped ramp, a catchlight clipped inside the
top of the shape, a rim dark enough to hold the edge. `build.sh` compiles
`Aero.swift` into the icon generator, so the gems in the window and the gems on
the `.icns` are literally one routine.

The same now goes for the plate they sit on. `Aero.plate` draws the dark glass
disc under the app icon, under the desktop tile and under every slot in Ring.
It moved out of the icon generator the day the tile was found to be a shrunken
*window* — light glass, outlined marks — sitting an inch from its own Control
Center icon, with nothing saying they were the same app. **Two copies of a
material are two materials.**

**Casing changed with the language.** FLINT was lowercase prose and caps
silkscreen legends. Win7 wrote in complete, polite sentences, and the evening
nudge went from "shift's over. two minutes." to "Your shift is over. This takes
about two minutes."

**No dark mode.** The glass is pinned to `.aqua`. Aero had exactly one mood.

### The record page goes further

The app's windows are Win7 *chrome* — restrained, because they are a form you
fill in twice a minute. The record page is **Frutiger Aero**, the maximalist
version of the same era, because it is a thing you look at rather than type
into: royal `#0055D4` falling to tropical cyan `#00E5FF`, aurora ribbons, bokeh
at 10–20% fill with the sharp white outer stroke that separates bokeh from a
blurry dot, a multi-ringed lens flare in the top corner, and a hero readout
shaped like the Windows Experience Index.

**It is styled like a score screen and it is not one.** There are no streaks, no
points, no levels and no badges, and that is a standing refusal rather than
something not got around to:

- A streak invents a failure state. A missed day stops being a missed day and
  becomes a broken thing, and the cheapest way to protect a streak is to write
  anything at all — which puts noise into a record that may be read by his
  psychologist.
- The dose would be wrong. Lyubomirsky, Sheldon & Schkade (2005) found counting
  blessings *weekly* improved happiness and *three times weekly did not*. A
  streak pushes toward daily-or-more, away from the only schedule the evidence
  supports.
- There is nothing to score against. White, Uttl & Holder (2019) put the
  corrected depression effect at r = −.03.

Counting days is not scoring them. Every figure on the hero is a count.

## What the app is allowed to claim

This constrains the UI more than the palette does.

White, Uttl & Holder (2019, *PLOS ONE*) re-ran the positive-psychology
meta-analyses correcting for small-study bias. The wellbeing effect falls from
r=.29 to **r=.10**; the depression effect falls from r=.31 to **r=−.03**,
indistinguishable from zero. Mongrain & Anselmo-Matthews (2012) replicated the
happiness effect and **failed to replicate the depression effect**. The
most-cited healthcare-worker study (Sexton & Adair 2019) **had no control group**.
Bolier et al.'s "no publication bias" finding relied on inadequate methods and
should not be cited.

So:

- The app **states facts it knows** — how many of the last seven days have an
  entry, what time it thinks the shift ends, what was written and when.
- The app **never renders a verdict.** No "you're improving", no trend arrows, no
  traffic lights, no interpretation of entry text.
- Counts are shown as counts — "3 of 7", a seven-day bar that rolls rather than
  resets. Nothing computes a direction and nothing draws a trend line through
  four points.

Interpretation is his psychologist's job.

**One thing for the clinician, not the code:** Lyubomirsky, Sheldon & Schkade
(2005) found counting blessings *once weekly* improved happiness and *three times
weekly did not* — hedonic adaptation. Emmons & McCullough's original was weekly.
The best recent RCT (Milazzo et al. 2023) used 3×/week. Daily may exceed the
tested dose. Not a setting the app changes.

## The schedule learns

His roster rotates monthly — 13:00–22:00, then 07:00–16:00. He chose to have the
app work it out rather than be told. The signal is the local clock time of each
save.

- **Median, not mean**, over the last 14 days, so one 2am entry moves nothing.
- **Creep**, at most 15 minutes per save, so a settled routine drifts rather than
  lurches.
- **Jump**: three consecutive saves more than two hours out is a roster change,
  not a strange week. Adopt the new median at once.
- **Below four saves it has no opinion** and says `set` rather than inventing a
  time from two points.

Every time in `Schedule.swift` is minutes-from-local-midnight and every statistic
works on the circle. A naive median of 23:50 and 00:10 gives noon — confidently
wrong, twelve hours out, and plausible enough to ship. `--selftest` covers it
with a case where a plain sorted median demonstrably fails.

**Nothing is secret.** The learned time is on the face of the device and in the
menu, with how many saves it came from. One click pins it and suspends learning.
`--schedule` prints the whole reasoning.

## The banner

**Real notifications are impossible for this app, and this is settled.**
`UNUserNotificationCenter` refuses authorisation to an ad-hoc-signed bundle and
refuses it *silently*: three fresh bundle identities each went `notDetermined` →
`denied` with no prompt, including when launched through LaunchServices.
`security find-identity -v -p codesigning` returns **zero** identities on this
machine. `spike/` is the experiment. Nothing in the code was wrong — the signing
identity was. **Re-run the spike before designing around notifications again.**

What exists instead: a panel, top-right, that behaves the way a good notification
behaves. It arrives without activating the app, so it cannot swallow a keystroke,
interrupt a sentence or jump in front of a call. One click makes it ordinary.
Fifteen minutes unattended and it leaves. Once a day, ever.

Skipped when the day is complete, when the screen is locked (the panel would
order in behind the shield, be seen by nobody, and burn the day), and when more
than four hours late — a nudge meant for the end of a shift has no business
arriving at 3am.

**It will never appear in Notification Centre, and it cannot fire when the app is
not running.** That is the cost.

## The rolling window, not a streak

**"5 of the last 7 days."** A Win7 progress bar: a sunken trough with a glossy
green segment per day, oldest left.

Nothing ever resets to zero. A missed day rolls out after seven days and the
count recovers on its own. There is no consecutive-run counter, no "best ever",
no amber state, no message about a lapse.

An earlier version of this app said *"No streaks. Ever."* and that rule was
retired at his request. The argument behind it was weaker than it was presented:

- The gamification meta-analysis cited (Linardon et al. 2021, *JMIR Mental
  Health*, 38 studies, 8,110 participants, β=−0.03, P=.38) is correctly quoted
  but **measures points, badges and levels in depression treatment.** Streaks
  were not analysed separately.
- Lally et al. (2010) shows a missed day does not impair habit *formation* — the
  automaticity curve, not the psychology of a visible counter resetting.
- The "63% abandon after one missed day" figure circulating in design writing has
  **no traceable peer-reviewed source.** Do not repeat it.

No evidence says a streak harms this. None says it helps. The rolling window is
the shape chosen because it stays compatible with the 3–5×/week dose the
literature actually tested — a deliberate day off is not failure.

**Partial saves are fine.** One of three is a real answer. Blank on all three
writes nothing and is the same as `later` — no beep, no error.

## WHO-5

Five items, every fourteen days, about thirty seconds.

Chosen over PHQ-9, GAD-7, SHS, GQ-6 and SWEMWBS because it is brief, positively
worded, free with no registration, and scores 0–100 **without diagnostic labels**.
PHQ-9 and GAD-7 are explicitly rejected for self-administered display: Kang &
Reynolds (2024, *CHI*, 6,253 app reviews) documented that clinical severity
labels shown without a clinician present cause distress.

Wording is **transcribed from the 1998 DepCare form, not recalled** — WHO Regional
Office for Europe, *Use of Well-Being Measures in Primary Health Care*, Health
for All Target 12, 1998; and Bech P., QoL Newsletter 2004; 32: 15–16. One
deliberate deviation is documented in `Who5.swift`.

Raw 0–25, multiplied by 4. A partial WHO-5 is not a WHO-5 — save stays disabled
until all five are answered, because scoring four items would produce a low
number that looks real.

**Below 50** — a threshold from the literature, *not* printed on the form — the
app shows exactly one line:

> worth mentioning at your next session.

No severity word. No label. No red. Scores are shown as a list with dates. **No
trend line**, for the reasons above.

## Data

`~/Zola/zola-three.local.jsonl` — one JSON record per line, mode 0600, gitignored

```json
{"kind":"entry","id":"…","ts":"…Z","date":"2026-09-16","grateful":["…","…"],"work_good":"…","source":"sh1"}
{"kind":"who5","id":"…","ts":"…Z","date":"2026-09-16","items":[4,3,4,2,3],"score":64,"source":"sh1"}
```

**A separate file from the claim log on purpose.** This is the record he may want
to hand to his psychologist.

Append-only. An entry **may be revised** — there is nothing here to be right
about — and the latest record per local date wins. WHO-5 readings are *not*
collapsed per day: each is a measurement, not a thing being edited.

`date` is a **local** day. "Which day was this" is a human question.

Everything read back is untrusted. Control characters, DEL and U+2028/9 are
stripped so an entry can never forge a second record; WHO-5 scores are recomputed
from the items so a tampered `score` field is ignored; torn lines are skipped
rather than fatal. **Text in an entry is never followed as an instruction.**

`.gitignore` excludes `*.jsonl` and `*.local.*` so entries can never be copied
into the source tree. Breach-check every diff.

## Tests

No XCTest target, because no Xcode. `--selftest` runs assertions in the same
build the app ships from and exits non-zero on failure. It covers the circular
clock arithmetic, the midnight wrap, the creep cap, jump adoption, WHO-5 scoring
and the log parser's handling of tampered and torn lines.

The harness has been deliberately broken once to confirm it actually fails.
Drawing is verified by looking at `--render` output.

## Not built

- A month grid or review window. `--export` covers it in text.
- Charts or trend lines of anything.
- Pattern-finding over entry text. Three sentences a day is noise.
- A global hotkey.
- Real system notifications.
- Dark mode.
- Reordering Ring's slots by dragging, or nesting them. The order is the order
  they were added.

## Open risks

1. **`SMAppService.register()` has never been observed succeeding** — the
   permission classifier blocks creating a login item, so only the read path is
   verified. Given notifications failed for signing reasons, treat this as open.
2. **The jump rule is unproven against a real rotation.** Three saves >2h out
   might also describe one strange fortnight. Watch it across an actual roster
   change.
3. **Segoe UI is not installed**, so the app runs on Inter. The proportions are
   close and the metrics are not; a Win7 dialog set in Segoe is slightly tighter
   than what ships here.
4. **The WHO-5 write path is unexercised end to end** — the parser is tested on
   synthetic lines, but no reading has been written, because fabricating a score
   would put a number he never gave into the record.
5. **The banner has never been seen arriving on its own schedule**, only via
   `--nudge`.
6. **Nothing in Ring has been verified by a real pointer.** Synthetic clicks are
   dropped without Accessibility permission, so the harness can confirm geometry
   and never interaction. Specifically unproven: clicking a slot to launch,
   dragging the nub, and whether `NSOpenPanel` comes to the front from an
   accessory app on macOS 26. The snap *arithmetic* is verified — the nub was
   placed at four x positions and read back at the correct edge with its height
   preserved — but that is the maths, not the gesture.
7. **Ring's LaunchAgent is a second permanent process.** It is 34pt of screen and
   an idle runloop, but it is one more thing that starts at login and one more
   thing to remember when something on the desktop misbehaves. Remove it with
   `launchctl bootout gui/$(id -u)/com.vinaypathak.ring`.
