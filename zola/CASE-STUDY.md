# ZOLA — Behance Case Study (ready-to-ship draft)

> This file is the whole artifact, minus the pixels. Everything below is written to be
> pasted straight into Behance. Anywhere you see **[VISUAL: …]** is a screenshot or
> screen-recording to capture from the running app. Ship-checklist is at the very bottom —
> it's ~20 minutes of capture + paste, not a project.
>
> PRIVACY: this draft deliberately contains **no** employer name, no real finances, no
> colleague names, no "exit" narrative. It presents ZOLA as a design + engineering object.
> Keep it that way when you publish.

---

## 0. PROJECT META (the Behance fields)

**Title:** ZOLA — a personal operating system

**Subtitle / tagline (pick one):**
- *A single-page instrument for running a life like a system.*
- *One screen. Every domain of a life, orbiting a core — with a decision engine that refuses to flatter.*
- *Design, motion, and a deterministic reasoning layer, fused into one living interface.*

**Tools:** Vanilla JavaScript · GSAP (motion) · Three.js / WebGL (the core scene) · SVG · a small Python companion server · Python reasoning engines · HTML/CSS design system (hand-built tokens)

**Fields / categories:** UI/UX · Interaction Design · Creative Coding · Information Design · Motion

**Tags:** creative technology, interaction design, generative UI, dashboard, personal software, WebGL, GSAP, design systems, data visualization, dark UI, reasoning engine, single-page app

---

## 1. THE COPY (section by section)

### ▸ Cover
**ZOLA**
*a personal operating system*

**[VISUAL: the hero shot — the full orbit at rest. Core "eye" lit, nodes arranged in their ring, the aura breathing behind them. Shoot at full-bleed, dark. This is the thumbnail; it has to earn the click.]**

---

### ▸ Opening statement (the hook — first paragraph a visitor reads)

We track our steps, our sleep, our screen time. We instrument everything except the thing that actually runs — the life itself: where attention goes, where money goes, what we're building, what we keep deferring.

ZOLA is an attempt to build that missing instrument. It's a single-page operating system for one life — every domain rendered as a node orbiting a central core, each node a small application in its own right, all of it sitting on a decision engine that's designed to tell the truth rather than flatter the owner.

It is also, deliberately, a proof of a way of working: design, engineering, and applied AI treated as one discipline rather than three hand-offs.

**[VISUAL: a clean wide shot of the whole interface, slightly annotated — call out "core", "node", "aura". Keep annotations minimal and lime.]**

---

### ▸ The premise (why build this at all)

Most personal dashboards are read-only — a wall of numbers you glance at and forget. The interesting problem isn't *displaying* a life; it's giving it structure and consequence:

- Every domain in one field of view, so nothing important stays out of sight.
- A shape that makes relationships legible — what feeds what, what's neglected.
- A layer underneath that can look at the whole state and say *this is on track* or *this is you avoiding the hard thing.*

So the brief I set myself: **not a dashboard — an instrument.** Something you operate, that pushes back.

---

### ▸ The concept — a tree of life, seen from above

The organizing metaphor is an orbit. A single core sits at the center — an "all-seeing" eye, the system's sense of the whole. Around it, the domains of a life orbit as nodes: **craft & career, capital, learning, creative practice, the work in flight, and a private vault for the decisions that don't belong anywhere else.**

Choosing an orbit over a grid was the key design decision. A grid says *these are separate tiles.* An orbit says *these all revolve around one thing, and they pull on each other.* The core isn't decoration — it's where the whole-system read lives.

**[VISUAL: close-up of the core "eye" — the iris/bloom detail. Then a second shot pulling back to show the nodes in orbit around it.]**

---

### ▸ The design language — "quiet superpower"

The whole thing runs on a small, strict visual system I built from scratch — no UI kit, no framework defaults. The goal was a feeling: *calm, dark, precise, a little bit sci-fi — an instrument, not an app.*

- **Near-black canvas.** Almost everything is drawn on deep near-black, so light means information.
- **One electric accent.** A single lime signal color for "live / now / act." Used sparingly, it always means something.
- **Aura gradients.** Soft colored light blooms behind key surfaces — the interface feels *lit from within* rather than filled with panels.
- **Frosted glass.** Layers read as depth: closer things are frosted and bright, deeper things recede.
- **Dot-matrix numerals.** Every important number is drawn as a custom dot-matrix glyph — a tiny renderer that plots digits as dots. It makes data feel *instrument-grade*, like a readout, not a font.

**[VISUAL: a tight grid of 4 crops — (1) the aura bloom, (2) a frosted-glass panel edge, (3) the dot-matrix numerals up close, (4) the lime accent in context. Caption each with the token name.]**

---

### ▸ The system — nodes as small apps

Each node opens into its own focused surface. They share the language but each solves a different problem:

- **Craft / career** — a self-scoring view of growth: skills mapped, a ladder, the reputation moves that actually compound. It resists vanity metrics — a skill only counts when it's shipped something.
- **Capital** — a live financial cockpit. Goals on one layer, moves on another, colour-linked; live market quotes feed a real-time value read and a projection. Goals compute their own required contribution to land on time.
- **Learning** — a personalized journey, a skill map with computed gaps and risk, and a plan where each goal is tied to a resource, a date, and a required artifact. Proof-gated: reading isn't learning until it produces something.
- **Creative practice** — a plan / track / analyze loop for an ongoing creative output.
- **The core** — reads every node and renders a single "state of the system": readiness, momentum, and the one thing most worth doing next.

**[VISUAL: 3–4 node panels, shot individually, full-bleed. Pick the most visually rich: the capital cockpit (gauges + live chart), the learning skill-map, the core read. One per slide.]**

---

### ▸ The differentiator — a deterministic reasoning layer

This is the part that makes ZOLA a *creative-technology* object and not just a nice UI.

Underneath the interface is a small reasoning engine written as **rules-as-code** — a blackboard architecture where each domain contributes a "part" to a shared verdict. It's fully **deterministic**: the same state produces the same read, every time. No model is called at runtime. Its whole job is to look at the system honestly and either say *proceed* or *you're avoiding the real move — here's the redirect.*

The design principle I kept coming back to: **a decision tool is only useful if it can disagree with you.** An engine that filters every input through a flattering conclusion just reproduces the thing it claims to fight. So the reasoning layer is built to ask sharp questions and gate on evidence, not to reassure.

I think of the pattern as **"deterministic core, agentic shell"** — a stable, auditable rule engine at the center, with the softer, generative help kept at the edges where it belongs.

**[VISUAL: the core's "command / architecture" view — the layered system map. If there's a terminal/verdict readout, screenshot it showing a real PROCEED / REDIRECT line.]**

---

### ▸ Motion & craft

Nothing in ZOLA hard-cuts. The boot sequence assembles the core from nothing; nodes settle into orbit with a staggered ripple; the aura breathes on a slow loop. The 3D core is a real WebGL scene — a small particle "neuron-cosmos" with bloom — composited behind an SVG/DOM interface, so the depth is genuine, not a picture of depth.

Every motion is doing a job: the boot earns attention, the stagger explains hierarchy, the breathing keeps the thing feeling *alive* and on.

**[VISUAL: this is where the FILM goes — the ~60s screen-recording. See section 2. Also drop one or two motion stills here.]**

---

### ▸ Closing / colophon

ZOLA is one person's operating system, but the interesting part was never the personal data — it's the argument the object makes: that design, engineering, and applied AI are one craft, and that the most honest interface is one that can push back.

Built as a single self-contained page. Every pixel, token, glyph, motion curve, and rule hand-authored.

*Design, engineering & system — [your name]. [year].*

**[VISUAL: end card — the wordmark ZOLA on black, the aura just barely alive behind it. Same frame that ends the film.]**

---

## 2. THE 60-SECOND FILM (script + shotlist)

**Concept:** a cold-open that assembles the instrument in front of you, tours it in four fast beats, and lands on the idea. No talking head. Text-on-screen + the interface + one music bed. It should feel like a device powering on.

**Format:** 1:00, 16:9 (also export a 9:16 vertical crop for IG/LinkedIn). Screen-recording only.

**Beat sheet (timecodes are targets):**

| Time | On screen (VISUAL) | Text-on-screen (short — 1 line) | Notes |
|---|---|---|---|
| 0:00–0:06 | Black. The boot sequence begins — core assembles from nothing. | *We instrument everything except the life itself.* | Let the boot breathe. No rush. |
| 0:06–0:14 | Core resolves; nodes ripple into orbit. Hold the full field. | *ZOLA — a personal operating system.* | The reveal. Title lands as the orbit settles. |
| 0:14–0:26 | Pan/zoom into the capital cockpit — gauges move, live chart draws. | *Every domain, one field of view.* | Pick the most alive node first. |
| 0:26–0:38 | Cut to learning skill-map, then the craft view. Quick, confident. | *Each node, its own instrument.* | Two nodes max — keep pace. |
| 0:38–0:50 | The core's reasoning read — the verdict line resolving. | *A decision engine that refuses to flatter.* | The differentiator beat. Slow down here. |
| 0:50–0:58 | Pull back out to the full orbit; aura breathing. | *Design. Engineering. A reasoning layer. One craft.* | The thesis. |
| 0:58–1:00 | Settle to the end card. | *ZOLA* | Silence + one last bloom. |

**Music:** one ambient/electronic bed with a soft build — nothing with vocals. Something that feels like a system coming online. Cut the music to end exactly on the end card.

**Capture notes:**
- Record at the highest resolution your screen allows; the near-black needs the bitrate.
- Do each beat as its own clean take (boot, each node, the verdict, the pull-back), then assemble. Much easier than one perfect run.
- Kill the cursor where you can; where you can't, move it with intent — it's a pointer, not a fidget.

**Assembly:** any editor works (CapCut / Premiere / DaVinci / the 1-min-AI tool). Text-on-screen in the lime accent or plain white, one weight, generous margins. No transitions except cuts and one slow fade to the end card.

---

## 3. IMAGE PLAN (the stills, in order, with captions)

Behance rewards a strong vertical scroll. Target ~8–10 images. Order + captions:

1. **Hero orbit at rest** — *ZOLA — a personal operating system.*
2. **Annotated whole-interface** — *One core. Every domain of a life, in orbit.*
3. **Core "eye" close-up** — *The core reads the whole system — not a logo, a sensor.*
4. **Design-language 4-crop grid** — *A hand-built visual system: near-black, one signal color, aura, glass, dot-matrix numerals.*
5. **Capital cockpit** — *Live financial cockpit — goals, moves, and a real-time value read.*
6. **Learning skill-map** — *Skills mapped to gaps and risk. Proof-gated: it only counts when it ships.*
7. **Core reasoning read** — *A deterministic decision layer. Same state, same verdict — and it's allowed to disagree with you.*
8. **Architecture / system map** — *Deterministic core, agentic shell.*
9. **Motion still (boot or orbit ripple)** — *Nothing hard-cuts. Every motion is doing a job.*
10. **End card** — *ZOLA. Design, engineering & system — [your name].*

Embed the **film** right after image 9 (the motion beat).

---

## 4. CROSS-POST COPY (reuse the same asset elsewhere — more surface, zero extra work)

**LinkedIn / long caption:**
> I built ZOLA — a personal operating system. One screen: every domain of a life rendered as a node orbiting a core, each node its own small app, all of it sitting on a deterministic reasoning engine designed to push back rather than flatter.
> It's also an argument I care about: design, engineering, and applied AI are one craft, not three hand-offs. Hand-built visual system, real WebGL core, rules-as-code decision layer.
> Full case study + a 60-second walkthrough → [Behance link]

**Instagram / short caption:**
> ZOLA — a personal operating system. One core, every domain in orbit, and a decision engine that won't flatter you. 60-sec walkthrough → link in bio. #creativetechnology #interactiondesign #webgl #uidesign

**Portfolio index one-liner:**
> ZOLA — a single-page personal operating system fusing interaction design, WebGL motion, and a deterministic reasoning engine.

---

## 5. SHIP CHECKLIST (~20 minutes)

- [ ] Capture the 10 stills in section 3 (open each node, screenshot full-bleed, dark).
- [ ] Record the film beats in section 2 (separate clean takes), assemble, add text + one music bed, export 16:9 + 9:16.
- [ ] Create the Behance project; paste the copy from section 1 section-by-section; drop each image at its marker.
- [ ] Set title, tags, tools, fields from section 0.
- [ ] Embed the film after image 9.
- [ ] Read once for any accidental private detail (employer, real numbers, names) — there should be none; keep it that way.
- [ ] Publish. Cross-post with section 4 copy.
- [ ] Log it in ZOLA as a shipped proof artifact.

That last box is the whole point: it's the one move that lifts readiness off the floor.
