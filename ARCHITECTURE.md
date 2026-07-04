# ARCHITECTURE — Sam Walks site

The walkable website: a side-scrolling, game-style page where the visitor walks a Lisbon dirt road while Sam talks in burned-in subtitle capsules. Built 2026-06-10 from the **Sam Walks Design System** handoff bundle (claude.ai/design export; reference copy in `../design-system/`).

- **Live**: https://sam-walks-site.vercel.app
- **Repo**: https://github.com/julesjewels-ai/sam-walks
- **Everything is one file**: `index.html` (~1,580 lines, ~62KB) + 3 photos in `assets/photos/` + `vercel.json`.

This document records what was decided, *why*, and how the build evolved. Companion to `README.md` (what it is / how to run it).

---

## 1. System overview

```
index.html
├── <head>     tokens (CSS custom properties), all styles, SEO/og meta, fonts via <link>
├── #boot      camera power-on loader (~1s, skippable, reduced-motion bypassed)
├── #title-screen   landing "the road remembers": hero, 2 caption capsules, road band,
│                   ● START WALKING, touch layers (#terrain footprints / #dust-layer), TC readout
├── #game
│   ├── #viewport > #world     the road: chapters, rocks, signs, km posts, 3 video stops, end zone
│   ├── #sam                   the walker (flat CSS figure; physics via JS)
│   ├── HUD                    ●REC + timecode, EP meta pill, INDEX button, watch hint
│   ├── #caption-stack         subtitle capsules (aria-live)
│   ├── #question-box          1/2 answer chips
│   ├── #touch-controls        ◀ ▲ ▶ (coarse-pointer media query + maxTouchPoints fallback)
│   └── #scrub-wrap            red scrub bar = position slider (keyboard-operable)
├── #index-overlay             dialog: episode list, teleport rows, direct watch links, Subscribe
└── <script>   data (STOPS/CHAPTERS/ROCKS/EVENTS) → world build → rAF game loop → controls
```

**Camera model**: Sam is fixed at 38vw (desktop) / 16vw (mobile) of the viewport; the `#world` div translates by `samScreenX() - samX`. One plane, one speed — the design system bans parallax, which conveniently makes the camera trivial.

**Data model**: four plain arrays drive everything.
- `STOPS` — the three published videos (real titles, durations, YouTube IDs from `data/videos.csv` in the channel repo).
- `CHAPTERS` — pillar headings painted in the sky.
- `ROCKS` — jumpable obstacles with one-take remarks.
- `EVENTS` — monologue captions and the two viewer questions, keyed by world-x; `fired` flags prevent replays. `teleport()` marks everything passed as fired so scrubbing never replays the monologue; the **Back to the start** button explicitly resets all flags because there it *means* "play it again".

---

## 2. Decision log

### D1 — Vanilla JS/DOM, no game engine
Phaser/Three.js/PixiJS were considered and rejected. The design system's motion law — *cuts not fades; no parallax; no blur; no bounces; polish is off-brand* — eliminates nearly everything an engine is for. A flat DOM side-scroller delivers the brand exactly, costs zero dependencies, loads instantly, runs from `file://`, and keeps the content as real text (links, headings, aria) rather than canvas pixels. Accessibility fell out almost for free because of this choice.

### D2 — One self-contained HTML file
No build step, no bundler. The site is small enough that splitting files only adds deploy/caching surface. Tokens are copied **verbatim** from `design-system/tokens/*.css` so the file is the single source of truth at runtime while the design system remains the source of record.

### D3 — Side-scroller with a visible Sam, not first-person
The brand is "I'm walking and I'm talking." First-person (you-are-the-camera) was considered and rejected: showing a figure that *walks* makes the format invariant legible in one glance. Sam is built from flat rectangles in palette colours (asphalt body, signal-red jacket stripe, dust head, white lav mic) — the design system bans illustration, and a blunt geometric figure reads as diagram, not illustration. Walk cycle and bob use `steps(2)` so even his gait is cuts.

### D4 — The red line is the scrub bar is the walk
The system's core motif (4px signal-red scrub line) is used *functionally*: the bottom red bar is the walk's progress AND the seek control AND (per the brand readme) "the view's one red element" in the game HUD. It is a real `role="slider"` — drag, click, arrow keys, Home/End, `aria-valuetext` as a timecode ("4:12 of 8:24"). The fake timecode maps walk progress onto 8:24 = the summed duration of the three real videos.

### D5 — Talking = burned-in subtitles, not speech bubbles
Captions render as the design system's CaptionBar (solid asphalt capsule, white bold text) fixed at bottom-centre **like subtitles on the videos**, not floating bubbles above the figure. Lines come from the real video descriptions and `brand/tone/tone.md` (British English, "guys", signposting, emphasis repetition). Lines cut in/out — no typewriter effect, no fades. Pacing 3s/line after usability feedback that 2.4s was rushed.

### D6 — Street interruptions as game mechanics
Rocks block the road and must be jumped; each fires a one-take remark ("Hop over — still one take."). This is the brand signature (street interruptions handled with humour) turned into the game's only obstacle. Deliberately no fail state, no score, no timer — the channel is anti-hustle; the game must not be a hustle. A rock remark is suppressed while a question is pending (audit-found bug: it used to erase the prompt).

### D7 — Questions are non-blocking
Sam asks real viewer questions ("What are you prioritising?"). Chips answer with 1/2 keys or tap; walking on past (>700px) silently drops the question, exactly like not answering a YouTuber out loud. The prompt caption persists until answered or dropped (was: auto-cleared at 6s leaving orphaned chips — fixed).

### D8 — Photos: inline base64 → lazy files (reversed decision)
History: photos started as files; were inlined as base64 data-URIs to survive a preview sandbox whose base path broke relative URLs; then **reverted to lazy-loaded 640px files once properly hosted**, because the inline payload was 85% of the page (297KB → 55KB) and billboards aren't visible at first paint. Lesson recorded: inlining was a workaround for a preview artefact, not an architecture; hosting removed the constraint.

### D9 — Fonts non-blocking, weights trimmed
Google Fonts `@import` inside CSS was render-blocking (HTML → CSS → font CSS chain). Replaced with `<link rel="preconnect">` + `<link rel="stylesheet">` and trimmed to the weights actually used (Archivo 400/700, Archivo Black, Space Mono 400/700). `font-display: swap` means font failure degrades to system stacks without breaking anything.

### D10 — Mobile-first geometry
Phones are the primary device (owner's call). Consequences:
- Sam anchors at **16vw** so centre-fixed subtitles never cover him (38vw desktop).
- Billboards shift toward viewport centre (`translateX(132px)`) because the stop's world-x centres on Sam, which on mobile would push the card off-screen left.
- Sam renders 1.35× via the CSS `scale` property — separate from `transform`, which the game loop owns for jump translation. Physics/collision are unscaled (visual-only enlargement).
- Touch controls ◀ ▲ ▶ are 72px, keyboard-operable, with `env(safe-area-inset-bottom)` so nothing hides behind the iPhone home indicator. Shown via `(hover: none) and (pointer: coarse)` plus a `navigator.maxTouchPoints` JS fallback for browsers that mis-report the media query.
- End zone width is computed from the space ahead of Sam (`calc(62vw - 110px)` desktop, `74vw` mobile) and sits *past* `WORLD_LEN` so you walk into it at the road's end (v1 placed it before the end — invisible on arrival).

### D11 — Accessibility posture: semantics over simulation
Because content is real DOM (D1), a11y is native semantics, not bolt-on: scrub = slider, index = `role="dialog"` with focus trap and focus return, captions = `aria-live="polite"` region (captions ARE the content for SR users), decorative scenery/figure/dust `aria-hidden`, video cards carry "Watch on YouTube: {title} ({duration})" labels, all targets ≥44px. Contrast: small mono text overrides the token (`ink-3` 3.2:1 → `ink-2` 7:1 on white; `ink-4` on asphalt) — the *token file stays verbatim*, overrides live at usage sites with comments, so the design system remains the record and the deviations are explicit. `prefers-reduced-motion` kills all animation; the game remains fully playable because movement is JS-driven position, not CSS animation.

### D12 — Boot loader = camera power-on (component-library audit)
Owner supplied 16 MagicUI components as candidates. Verdict: **14 rejected** — ripple, aurora/gradient/dia text, animated-gradient text, rainbow + pulsating buttons, cool-mode particles, kinetic/hyper text (as-is), scroll reveal, hero video dialog, theme toggler, video-in-text, circular progress — all collide with brand law (gradients banned; fades banned; decorative loops banned — the blinking REC dot is "the one permitted loop"; the progress motif must be the red *line*, not a circle). **2 adapted, rebuilt from scratch in brand language** (~60 lines total, no libraries):
- *Loader* → ~1s boot: asphalt screen, blinking ●REC, timecode ticking 0:00→0:04, red line growing in 16px steps, hard cut to title. Skippable on any input; skipped entirely under reduced motion.
- *Hyper-text scramble* → title meta line settles like a timecode locking in: hard character swaps at 55ms, 7 iterations, one-shot, no easing. *(Retired in D16 — the meta line it animated was cut from the landing.)*
Hero lines cut in like subtitles switching on (`steps(1)`, 0.05/0.25/0.45s delays). The hiding comes *only* from `animation-fill-mode: both` pulling the keyframe's `from { visibility: hidden }` state during the delay — there is deliberately no static `visibility: hidden` rule, and JS only adds the gating `.anim` class when motion is allowed. So under `prefers-reduced-motion` (no class, animations stripped) the text's computed visibility is `visible` — verified empirically. That interaction is subtle and load-bearing; don't "simplify" it by adding a static hidden state, which would blank the hero for reduced-motion users.

### D13 — Landing: minimal but not bare *(superseded by D16)*
One screen on mobile: wordmark, red rule, mono meta (scramble), three-line hero, four-line lede in Sam's voice (includes duration honesty — "a couple of minutes"), two full-width CTAs. **Jump to a walk →** starts the game with the index dialog open — two taps from cold to any video, the conversion shortcut the usability audit demanded. Keyboard hint chips render only on pointer devices; touch users learn from the road signs and captions.
Owner verdict after living with it: still confusing — two CTAs unnecessary, hint chips are instructions, the lede is a wall of text. Replaced wholesale by D16; the in-game Index button and road signs already cover everything the second CTA and the chips did.

### D14 — Hosting: GitHub + Vercel CLI, deliberately not linked
Repo scoped to `creativity/sam-walks-site/` only — the channel repo contains OAuth credentials that must never travel. Vercel deploys via CLI (`vercel deploy --prod --yes --scope isas1s-projects`); Git pushes do **not** auto-deploy. Trade-off accepted: manual deploys keep the credentialed parent directory entirely out of any CI surface; connect the repo in the Vercel dashboard later if push-to-deploy is wanted. `vercel.json` adds nosniff / referrer-policy / frame-ancestors (static site, no inputs — CSP kept minimal on purpose). Caching is Vercel's default `max-age=0, must-revalidate` + ETag: correct for one mutable HTML file.

### D15 — SEO/social surface
All game content is JS-built, so crawlers/unfurlers get: full og/twitter card set, canonical, a real favicon file set (`favicon.ico` + `favicon.svg` + PNG/apple-touch sizes under `assets/icons/`, the red square frame mark — moved off a data URI so Google Search and social unfurlers can actually fetch and cache it), one unified meta description matching the lede across `<meta name="description">`/og/twitter, and a `<noscript>` block with the three video links + subscribe. The title screen itself is server-rendered HTML, so there is real content without JS too.

### D16 — Landing v3: "The road remembers" (design-panel synthesis)
Owner brief: one CTA only, no instructions, no wall of text, curiosity-driven, touch-alive, "awwwards site of the year". A 4-designer/3-judge/1-synthesis panel produced four concepts (camera title card / street poster / persistent road / recording road); judges split 1-1-1, but two of three put a road-world concept first, and both called **footprint persistence into the game** the strongest idea in the field. Synthesis: spine = *The road remembers*, grafts = whole-screen dust (so every touch answers), pointer-down-only TC roll, caption-capsule support copy, tap-always-stamps + one single-use "SUBJECT: YOU" microlabel.

What shipped:
- **The title screen stands on the game's own road.** The road band is built from the same tokens as the in-game ground (`--ground-h` 148px, 7px `--field` strip, 2px ink horizon), so the menu→game transition is a cut where the horizon does not move a pixel — "NO CUTS" made literal. This geometry is **load-bearing**: change `--ground-h` and both surfaces move together or the payoff dies.
- **Touch system, two materials split at the horizon** (`innerHeight − 155`): above it, flat dust squares (24-grid quantised, sizes 14/10/8 cycling, every 4th ink) that settle via `steps(3)` scale-cut in 450ms; on the road, 9×16 boot prints (heading snapped to 15°, alternating ±7px per boot, kick squares hopping out via `steps(3)`/180ms) that **persist**. On game entry the `#terrain` layer is re-parented into `#world` with a `-samScreenX()` left offset compensating the world transform at `samX = 0` — verified: identical screen x-positions before/after the cut. Wrapped in try/catch as progressive enhancement.
- **Pools, not allocation**: 24 dust + 16 feet + 12 kicks = 52 divs created once at boot, recycled oldest-first (a cut, never a fade); pointermove is rAF-gated; zero runtime `createElement`.
- **One CTA**: `● START WALKING` — an asphalt caption capsule standing on the road in the thumb zone. The blinking red dot is simultaneously the view's only red element and its only permitted loop (the wordmark's red square is demoted to ink on this view to keep the budget). The ▶ glyph died: it promised video playback; the button starts a walk.
- **Copy budget ~11 words**, two caption capsules ("Sam talks — AI, quitting, life abroad." / "You do the walking.") that pre-teach the game's subtitle language. Everything else died: second CTA, five kbd hint chips (desktop too), 40-word lede, meta line + scramble, footer wordmark, red rule.
- **Camera-speak**: `TC 00:00.0` top-right rolls *only while a pointer is down* and freezes on release, accumulating across touches — the thesis as a mechanic. Coordinates stamp the road under the horizon. No aria-live on either (decorative).
- **Rejected from the panel's own output**: ghost-walk idle demo (a page preaching stillness must not perform unprompted), 60ms shutter frame on entry (inserts a cut at the exact moment the design proves there is none), seeding the in-game timecode (it is position-derived — `fmtTime(p * WALK_SECONDS)` — an offset would corrupt the `/ 8:24` mapping), crop marks, coordinate ticking, a "scuff the road" hint chip (instruction text the owner asked to remove; the first accidental touch teaches it).
- `touch-action: pinch-zoom` on the title (not `none`): one finger draws, two fingers still zoom — keeps WCAG zoom intact. Under `prefers-reduced-motion`, stamps appear instantly (the global animation kill strips the steps hops; dust falls back to a 450ms timer cut), kicks are skipped, and the TC shows whole seconds.
- Hero cut-out on start sets `animation: none` inline *before* `visibility: hidden` — the `.anim` cut-in's `fill-mode: both` would otherwise override the inline visibility (same subtlety as D12, in reverse).

---

## 3. Update log

| Commit | What | Driver |
|---|---|---|
| `8055883` | Initial build: world, Sam, captions, 3 real-video billboards, scrub-seek, index overlay, end zone; jump physics, rocks, dust puffs, walk cue; mobile pass 1; inline base64 photos | Design-system handoff + "make it a walk" concept; live-test feedback (pacing, direction cue, jump) |
| `d1bc35b` | `.gitignore` (`.vercel`, `.DS_Store`) | First Vercel link |
| `a516fb7` | Audit round: photos→lazy files, fonts→link, iOS scrub pointer-capture + `touch-action`, Enter double-open guard, rock/question collision, back-to-start replay, TDZ fix, aria-valuetext, keyboard touch-buttons, index watch links + Subscribe, scrub EP labels, SEO/og/noscript, `vercel.json` headers, caption pacing 3s | Three-agent audit of the live site (QA / accessibility / usability). Headline finding: all three YouTube links were private at launch — scheduled publishes Jun 11–13, self-healing, not a code bug |
| `a04fde3` | Landing redesign: boot loader, hard-cut hero, meta scramble, trimmed lede, Jump-to-a-walk CTA, kbd hints hidden on touch, end-zone/HUD collision fix | Owner's Android screenshots + "minimal but not bare, intro loader, stay on brand"; component-library audit (D12) |
| `75c024d` | README + meta description synced to shipped state | Docs drift |
| `1dbb643` | ARCHITECTURE.md added — decision log, update history, constraints | Owner: "keep track of your updates and why in an architecture.md" |
| `a1c0f8b` | Landing v3 "the road remembers": single CTA, ~11-word copy, touch-reactive road (dust/footprints/TC), footprints persist into game; kbd CSS removed | Owner: "one CTA, no instructions, curiosity-driven, awwwards landing, things happen on touch" + design panel (D16) |

---

## 4. Known constraints & lessons

- **Video links dead until 2026-06-13T16:44Z** — billboards point at scheduled-private videos (24h veto window staggering). External; self-heals.
- **No auto-deploy** (D14). Redeploy command lives in README.
- **Older Safari (<14.5 / <15.4)**: CSS `scale` property and `steps(…, jump-none)` silently degrade (small Sam / gliding legs). Accepted for 2026 traffic; fallback would be moving scale into the `.flip` wrapper transform.
- **WORLD_LEN = 9200** (tightened from 10400 — first video reachable in ~6s). World-x values in STOPS/CHAPTERS/EVENTS/ROCKS are hand-placed against it; change it and re-space everything.
- Process lessons (for future sessions, recorded in project memory too): the preview panel caches hard — restart the preview server after big rewrites; Python `str.replace` edits no-op silently when the target string wraps lines — prefer exact-match Edit tooling; headless preview throttles rAF and timers, so physics were verified by stepping `loop(t)` with synthetic timestamps.

## 5. Extending

- **New episode**: append to `STOPS` (x, ep, title, place, dur, img, url) and add caption `EVENTS` around its x; extend `WORLD_LEN` by ~2,500px per stop and move the end-of-road `CHAPTERS` entry + scrub ticks rebuild automatically. Photos: 640px JPEG in `assets/photos/`.
- **Metadata pipeline integration** (future): `STOPS` could be generated from `data/videos.csv` at deploy time; deliberately not done yet — three hand-placed stops didn't justify a build step (D2).
