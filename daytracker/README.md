# daytracker — phase 1: the logger

A permission-free macOS activity logger. Polls every 30s, attributes elapsed
time to the frontmost app, and refuses to attribute time it cannot vouch for.

Phase 1 is **the logger only** — no dashboard, no goal-linking, no capacity
engine. The capacity engine needs an EWMA with a 2–3 week half-life and an
acute:chronic ratio over a trailing 28 days; neither can be calibrated against
guesses. So the logger ships first and starts accumulating. Revisit the engine
once there is a month of real data.

## The privacy boundary

Captured, per contiguous app session: **app display name, bundle id, duration,
category.** That is all.

Never captured, stored or logged: **window titles, document names, file paths,
URLs, clipboard content.**

This is structural, not a promise. Window titles on macOS require Accessibility
permission. This program shells out to exactly four things:

| command | returns | permission |
|---|---|---|
| `lsappinfo front` | frontmost app's ASN | none |
| `lsappinfo info -only name <ASN>` | `LSDisplayName` | none |
| `lsappinfo info -only bundleid <ASN>` | `CFBundleIdentifier` | none |
| `ioreg -c IOHIDSystem` | `HIDIdleTime` | none |

None can return a window title, so the boundary cannot be crossed by accident.
The `-only` flags are deliberate — a bare `lsappinfo info` dumps a much larger
record. **Do not widen them.** If a change appears to need Accessibility
permission, stop and ask first.

Browsers are one `browsing` bucket on purpose. Splitting by site would require
URLs, which is exactly what this boundary forbids.

Audit it yourself:

```sh
grep -rniE 'https?://|/Users/|\.docx|inbox|untitled' ~/tools/daytracker/data/
```

## Layout

```
~/tools/daytracker/
  daytracker.py                       the logger
  categories.json                     bundle id -> category, yours to curate
  com.vinay.daytracker.plist.example  LaunchAgent template — see "Running it
                                      under launchd" below for current state
  data/
    activity-YYYY-MM-DD.jsonl         raw append-only log, one file per day
    summary/summary-YYYY-MM-DD.json   daily rollup, per category and per app
    .state.json                       last poll, for --status
    daytracker.log                    operational log
```

Daily files mean a corrupt or oversized file costs one day, never the history.
A single unparseable line is skipped, not fatal.

## Commands

```sh
python3 daytracker.py                 # run the logger (Ctrl-C to stop)
python3 daytracker.py --watch         # live view; one frame if piped
python3 daytracker.py --set-shape blue   # declare the current ZOLA lane
python3 daytracker.py --shape         # print the declared lane
python3 daytracker.py --once          # print current app / category / idle
python3 daytracker.py --status        # what the daemon last saw
python3 daytracker.py --unsorted      # bundle ids still needing a category
python3 daytracker.py --rollup        # rebuild all summaries + ZOLA rollup
python3 daytracker.py --selftest      # idle / sleep-wake logic tests
```

`--poll`, `--idle-threshold`, `--max-gap` and `--duration` override the defaults
(30s / 300s / 120s) and are mainly for testing.

## How time is accounted

Every second between the first and last record is classified. Nothing is
invented and nothing silently vanishes.

- **session** — the frontmost app, uninterrupted. A switch closes one and opens
  the next.
- **gap / `idle`** — no HID input for 5 minutes. The session end is *backdated*
  to when input actually stopped, not when the poller noticed.
- **gap / `untracked`** — consecutive polls more than 120s apart *and* input
  occurred during the gap. The machine was awake and in use but we were not
  watching, so we cannot know which app. **This is the rule that stops 8 hours
  of overnight sleep landing on whatever app was in front when the lid closed.**
- **gap / `idle` (after downtime)** — polls far apart but the idle clock covers
  the whole gap, so nobody touched the machine. Honest to call that idle.
- **gap / `screen-locked`** — frontmost is `loginwindow` or the screensaver.
- **gap / `no-frontmost`** — the frontmost app was unreadable. Recorded as
  unknown rather than guessed.

Sessions crossing local midnight are split at the boundary so daily files stay
clean; the halves carry `split` / `split_of`. Any span longer than
`MAX_SPLIT_DAYS` (2) is truncated to its tail and marked `truncated_from`, so
one absurd interval can never write a file per intervening day.

A single instance is enforced with `flock` — two daemons would double-count.

## Schema

```json
{"v":1,"type":"session","start":"2026-09-16T15:41:25+05:30",
 "end":"2026-09-16T15:41:56+05:30","seconds":30,
 "app":"Calculator","bundle_id":"com.apple.calculator","category":"unsorted"}

{"v":1,"type":"gap","start":"...","end":"...","seconds":48,"reason":"screen-locked"}
```

Timestamps are local ISO-8601 with offset, parseable by
`datetime.fromisoformat`. The nine record keys are the whole vocabulary:
`v type start end seconds app bundle_id category reason`.

The per-day summary is what the later engine should read — 28 trailing days
costs 28 small file reads, never a scan of the raw log.

## Categories

`categories.json` maps bundle id → category, **exact match only**. Unknown apps
become `unsorted`, keeping their real name and bundle id so no time is lost.
daytracker never guesses a category: a wrong guess is worse than an honest
`unsorted` you can see and fix. Run `--unsorted` for the curation queue, sorted
by time. The file is re-read on every poll — no restart needed.

## ZOLA hand-off

One file, `~/Zola/zola-day.local.json`, holding a rollup and nothing else:
total tracked hours and the top three categories, plus a prebuilt `line` for
ZOLA to render. **No app names, no bundle ids, no sessions, no timestamps beyond
the date.** ZOLA never receives the raw log — detail stays in the tracker. It is
gitignored by `~/Zola/.gitignore`'s blanket `*` allowlist; verify with
`git check-ignore -v zola-day.local.json`.

It prefers *yesterday* (a complete day, so the figure does not creep upward all
afternoon) and falls back to today when there is no yesterday. `line_source`
says which.

## The reporting window — what FORGE reads

The hand-off used to carry yesterday and today only. It now also carries
`windows`: one aggregate per span in `WINDOW_DAYS` (7 and 28), each with
`tracked_hours`, `tagged_hours`, `attributed_hours`, `tagged_share`,
`away_hours`, `untracked_hours`, `gaps_unlabelled`, the `focus` /
`categories` / `activities` splits, a compact `by_day`, and a `readout`.

`SCHEMA_VERSION` deliberately does **not** move. It is shared with the `v` on
every daily JSONL record, and bumping it here would claim the record format
changed when only the rollup gained a key.

Three deliberate choices:

**`days_with_data` travels with every window.** A 7-day window holding two days
is not a week. The panel says "2 of 7 days have data" instead of drawing a
week-shaped chart from a Tuesday and a Wednesday.

**`tagged_share` divides by `attributed_hours`, not `tracked_hours`.** They
differ: a labelled away-gap (a meeting) carries a project without ever being
desk time, so the focus totals can exceed tracked time. Dividing by tracked
made the panel's own percentages add up to 104%.

**`readout` is rule-based prose, and it is allowed to be flat.** There is no
network access here, so there is no model to ask — which is a feature, because
a rule can be audited and cannot embellish. It picks at most three clauses by
priority and drops any it cannot support. Two priorities are load-bearing: when
most time carries no project, *that* leads instead of a top project; and when
the logger was off longer than it was on, that caveat outranks the gap nudge.
50 unlabelled gaps is true, unactionable, and hides the reason they exist.

### Retired labels

`projects.json` takes a `retired` list. Its hours are counted as `unset`
because that is what they were — time with no project named. The current
entries (`black`, `blue`, `white`) are the coarse lanes this tool used before
projects existed, and they were **ZOLA node names**, not projects.

The fold happens in `fold_retired()`, which every surface reads through, so the
menu bar, the rollup line and the FORGE panel cannot disagree about whether
`black` is a project. Nothing on disk is rewritten: delete a name from the list
and its hours come straight back. A name in `projects` outranks its own
retirement, so reviving one takes a single edit.

## Footprint — measured, not estimated

All figures measured on this machine, 2026-09-16. Re-measure rather than trust
this table if anything changes.

| | |
|---|---|
| CPU, logger | ~92 s/day total, i.e. **0.1% of one core** |
| CPU, TRC | idles at 0.0%; a 5 s tick that reads two small files |
| Memory, logger | ~13 MB RSS |
| Memory, TRC | ~43 MB RSS (the AppKit baseline; unavoidable for any Cocoa app) |
| Disk | **54 KB/day**, ~19 MB/year, at 184 bytes/record and 300 records/day |
| Daily summary | 1.2 KB |
| File writes | `.state.json` once per poll; the day log only on a real event |

Nothing is ever loaded whole: daily files cap the working set at one day, and
the later engine reads the small summaries, not the raw log.

### Three wasteful things that were found and fixed

Measurement, not intuition, found all three — the first two were invisible
without looking:

1. **`ioreg` was piping 390 KB per poll** to read a single integer: about
   **1.04 GB of text parsed per day**. `-r -d 1` roots the dump at the
   IOHIDSystem node and returns 4 KB for the same value — **95× less**, and
   verified to produce identical readings. Poll CPU fell ~42%.
2. **TRC rewrote `.menubar.json` every 2 seconds** — 43,200 writes/day of an
   almost always identical file. It had burned **7× the CPU of the logger it
   was merely watching**. Now it writes only when the display changes.
3. **The ZOLA rollup was rewritten every 5 minutes.** It holds one summary line
   per node, so it is now hourly.

Note `ioreg -k HIDIdleTime` looks like the obvious fix and is **wrong**: `-k`
filters which *nodes* print, not which keys, so it returns a node header with no
properties and the idle parse silently yields nothing. That would have disabled
idle detection without any error.

## ZOLA lanes — declared, never inferred

Time is tagged with one of three ZOLA shapes: `white` (employment), `blue`
(creative lab), `black` (civic & power) — or `unset`.

**This cannot be derived from the app.** `Terminal`, `Claude`, `Chrome` and
`Safari` each serve all three lanes depending on what is happening inside them,
and the only signals that would disambiguate are window titles and URLs, which
the privacy boundary forbids. So the lane is **declared** — you set it, the
logger stamps it. The app never guesses which lane you were in.

Declare it from the TRC menu bar, or from the shell:

```sh
python3 daytracker.py --set-shape white
```

Three consequences worth knowing:

- **A lane change splits the session.** Switch white → blue while staying in
  Chrome and the current session closes and a new one opens. A session has to be
  single-valued in shape or neither lane total is true.
- **`unset` is always visible** in the summary, the watch view and the ZOLA
  line. Untagged time that hides itself would let the lane totals silently
  overstate their own completeness.
- **Gaps carry no lane.** Time not attributed to an app is not attributed to a
  lane either.

`data/shape.json` is the interface between the menu bar and the logger. It is
re-read every poll, so a change lands within one poll with no restart.

## Focus — what you're on, declared

Each session carries a **focus**: a project label you declare. `work`,
`1minai`, `farming`. Free text, lowercased and trimmed, bounded at 40 chars.

This replaced the earlier three-lane White/Blue/Black split, because a project
implies the lane but a lane can never imply the project — and the project is
the unit a year-end review actually needs.

**It cannot be inferred.** Two hours of Chrome and Excel could be one client or
three, and the only signals that would disambiguate are window titles and URLs,
which the boundary forbids. So you declare it; the tracker never guesses.

```sh
python3 daytracker.py --set-focus 1minai   # or pick it in the TRC menu
python3 daytracker.py --projects           # the list TRC offers
```

`projects.json` is the list the menu offers — yours to curate, re-read live.
Removing a project there does **not** remove it from past records.

A focus change splits the session, exactly as an app switch does: a session must
be single-valued in focus or neither project total is true.

### `focus_source` — provenance that cannot lie

| value | meaning |
|---|---|
| `declared` | you set it, from the menu or the CLI |
| `rule` | a `focus_defaults` entry you authored set it |
| `unset` | nothing set it — and it was **not** guessed |

A declaration always outranks a rule, and a rule is never laundered as a
declaration. ZOLA can weight `declared` above `rule` rather than treating every
figure as equally trustworthy.

## Away-from-keyboard time

The tracker measures **screen attention, not work.** An hour in a meeting room,
a phone call, or thinking on a walk all read as `idle` — indistinguishable from
a coffee break. Measured on a real day, **52% of elapsed time was
unattributed.** For meeting-heavy work that is most of the job, so a
screen-only total systematically under-reports.

The fix is the same principle as focus: you say what it was.

```sh
python3 daytracker.py --gaps    # unlabelled gaps over 10 min
python3 daytracker.py --label-gap '2026-09-16T16:30:50+05:30' meeting --label-focus work
```

Activities are a **closed set** — `meeting`, `call`, `thinking`, `break`,
`commute`, `other`. A free list rots into synonyms and nothing downstream can
aggregate it.

Gaps under 10 minutes are never offered: nobody recalls what a four-minute gap
at 14:20 was, and asking would only train you to dismiss the prompt.

### Labels append, they never edit

The log is append-only, so a label is a record that *points at* a gap:

```json
{"v":3,"type":"label","target_start":"2026-09-16T16:30:50+05:30",
 "activity":"meeting","focus":"work","at":"..."}
```

`rebuild_summary` applies them, and the **latest label for a target wins** — so
a correction is just another append, the original stays on disk, and the whole
history is auditable. A labelled gap only then counts as activity; until you
label it, it stays honestly unaccounted.

## Guardrails — `rules.json`

The division of labour that keeps this honest:

> **The logger records facts plus provenance. ZOLA interprets.**

Rules live here, not in ZOLA, because rules applied where data is *written* are
versioned, testable and auditable. Rules applied downstream are invisible.

```json
{
  "min_dwell_seconds": 60,
  "lane_defaults": { "meetings": "white", "comms": "white" },
  "core_hours": { "start": "09:30", "end": "18:30", "days": [0,1,2,3,4] }
}
```

Re-read every poll. Edit freely, no restart.

### `min_dwell_seconds` — why most records were noise

Measured on a real day: **60% of sessions were under 60s but held only 25% of
the time.** Alt-tab churn was dominating the file.

An app must now hold the front for `min_dwell` to earn its own record. A shorter
glance:

- gets **no record** of its own, and
- does **not** split the session it interrupted, and
- is **not** credited to the app you returned to.

Its seconds become `churn_seconds` on the surrounding session. So a record stays
contiguous while the invariant holds:

```
(end - start) == seconds + churn_seconds
```

Time still reconciles exactly; it simply stops inflating whatever you tabbed
back to. `churn_seconds` in the daily summary is a fragmentation signal in its
own right — the cost of context-switching, which is precisely what the later
capacity engine wants.

### `lane_defaults` and `lane_source` — rules that cannot lie

A rule may *fill* a lane you did not declare. Every session therefore records
**how** its lane was set:

| `lane_source` | meaning |
|---|---|
| `declared` | you set it, from the menu bar or the CLI |
| `rule` | a `lane_defaults` entry you authored set it |
| `unset` | nothing set it — and it was not guessed |

A declaration always outranks a rule, and a rule is never laundered as a
declaration. ZOLA can then weight `declared` above `rule` instead of treating
every lane figure as equally trustworthy.

Keep `lane_defaults` to categories that are genuinely unambiguous. `meetings`
and `comms` are safe. Never map `browsing` or `build` — those are exactly the
categories that serve all three lanes.

### `core_hours` — flags, never assigns

Reported in the summary as `lane_outside_core_hours` so ZOLA can *ask*
("you logged Black at 11pm Tuesday — right?") rather than silently deciding.
The logger does not act on it. A guardrail that quietly rewrote your lanes would
be a guess wearing a rule's clothing.

## TRC — the menu-bar app

`TRC/` is a hand-assembled Swift menu-bar app (`LSUIElement`, no Dock icon).
Build it with `./TRC/build.sh` — Command Line Tools only, no Xcode.

It carries the live current app, the lane selector, today's lane split, and it
**spawns and supervises the logger**, which is why it replaces the LaunchAgent:
a menu-bar app becomes a login item via `SMAppService`, so there is no plist to
install and no `launchctl` to argue with.

The status item has three states, and the third is the point:

| Logger | Status item |
|---|---|
| tracking, lane declared | `◆ Claude` |
| away (idle / locked) | `◆ — idle` |
| **not polling** | `○ not logging` |

Filled `◆` means a lane is declared, hollow `◇` means `unset`, and `○` means the
logger is dead. Those are three different problems and they do not share a
glyph. The stale threshold is `2 × poll + 5` seconds, identical to the terminal
`--watch` badge, so the two surfaces can never disagree about when the logger
died.

It does not check whether a logger is already running before starting one. The
logger's own `flock` makes a second instance exit 1, so TRC treats an immediate
exit as "already running" rather than adding a second guard that could disagree
with the first.

### Verifying TRC without a screenshot

`screencapture` is whole-screen only, and photographing this display to prove a
menu bar works would pull Outlook and Teams content into a log — the exact thing
this project refuses to do. So everything is checkable headlessly:

```sh
TRC/build/TRC.app/Contents/MacOS/trc --selftest       # title + staleness logic
TRC/build/TRC.app/Contents/MacOS/trc --print-title    # what it would show now
TRC/build/TRC.app/Contents/MacOS/trc --login-status    # what SMAppService says
cat data/.menubar.json                                    # what it IS showing
```

`.menubar.json` is a heartbeat of the live display, written every refresh.
`--login-status` reports what the system says, never what was requested —
`zthree` learned that from notifications failing silently.

### The DMG — `TRC/package.sh`

`./TRC/package.sh` builds `TRC/dist/TRC-<version>.dmg`: a read-only compressed
image holding `TRC.app`, a symlink to `/Applications` to drag onto, and a plain
`READ ME.txt`. 168 KB. No installer script, no Finder window styling — nothing
that would need Automation permission to build.

The app inside is **self-contained**: `build.sh` copies `daytracker.py` into
`Contents/Resources/`, and `Paths.script` prefers that bundled copy over the
working copy in `~/tools/daytracker`. So TRC and its logger version together and
a new app can never end up driving an old logger. Confirmed live:

```
14151 .../TRC.app/Contents/MacOS/trc
14153 .../Python .../TRC.app/Contents/Resources/daytracker.py --exit-with-parent
```

This is safe only because `daytracker.py` derives `BASE` from `$HOME`, not from
`__file__`. A script running from inside a read-only mounted volume still reads
and writes `~/tools/daytracker` — it never tries to write next to itself.

**The DMG is not notarised and cannot be.** There are zero codesigning
identities on this machine, so both the app and the image are ad-hoc signed.
Locally built, there is no quarantine flag and it opens normally; copied to any
other Mac, Gatekeeper will block it until a right-click → Open. It is an install
medium for this machine, not a release.

Why bother at all: `SMAppService` refuses to register a login item for a bundle
sitting in a scratch `build/` directory, which is why `--login-status` still
reports `login item: not found`. `/Applications` is the location launch services
treats as a real install. **The drag step is yours** — the agent harness blocks
writes into `/Applications`, so this was verified by mounting the image,
`codesign --verify --deep --strict` (OK), and unmounting, not by installing.

## The `--watch` view

A live readout of what the logger is doing right now: whether it is polling, the
current app and category, today's split by category, and — deliberately — the
time it is *refusing* to count.

It extends the ZOLA Nothing law rather than inventing a second one: near-black
surface, a single accent (one constant, `ACCENT`, at the top of the watch
section), 1px hairlines, no gradients or shadows. Density and motion carry it.

The header badge is the important part. It reads `● LIVE` only when the last
poll is within `2 × poll + 5` seconds, otherwise `○ STALE <age>` or
`○ NEVER RUN`. A dead daemon looked alive for the better part of an hour during
development because nothing on screen contradicted the assumption — this badge
is the fix. Trust it over any memory of having started something.

This is not the dashboard. It reports the logger's own state, not analysis of
your time; the capacity engine and its UI stay out until there is a month of
data to fit them to.

## Running it under launchd

The plist is validated (`plutil -lint` OK) and the exact invocation launchd will
use has been dry-run successfully. Install and start it:

```sh
cp ~/tools/daytracker/com.vinay.daytracker.plist.example \
   ~/Library/LaunchAgents/com.vinay.daytracker.plist
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.vinay.daytracker.plist
```

Confirm it is actually up — do not assume:

```sh
launchctl print gui/$(id -u)/com.vinay.daytracker | head -20   # state, pid
python3 ~/tools/daytracker/daytracker.py --status              # last poll, current app
cat ~/tools/daytracker/data/launchd.err.log                    # must stay empty
```

`--status` should show a `last_poll` within the last 30 seconds and a
`current` app. If `launchd.err.log` is non-empty, something is wrong — read it
before trusting any data.

Stop and remove:

```sh
launchctl bootout gui/$(id -u)/com.vinay.daytracker
rm ~/Library/LaunchAgents/com.vinay.daytracker.plist
```

On older macOS, `launchctl load -w` / `unload -w` replace
`bootstrap` / `bootout`.

Expect one `untracked` gap covering whatever time passed between the last manual
run and the first launchd poll. That is correct: the logger refuses to attribute
time it was not awake for.

## Device policy — open question, unresolved

Nothing here was built to evade any of it, and nothing should be. Treat a
persistent LaunchAgent as a policy question to answer **before** installing the
plist. Running it in the foreground by hand raises no such question.
