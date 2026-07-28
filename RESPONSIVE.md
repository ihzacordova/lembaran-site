# Responsive audit — lembaran-site

**Audited:** `index.html` (1,773 lines, inline CSS/JS, no build step) and `privacy.html`, at
commit `78c6f69`. Verified as the deployed artefact first: `git remote -v` →
`ihzacordova/lembaran-site`, GitHub Pages source `main:/`, and the bytes served from
https://ihzacordova.github.io/lembaran-site/ hash-match the local `index.html`. (Safari hides
the `/lembaran-site/` path in its URL bar, which is why the address looks like the bare
domain.)

**Method:** headless Chromium, stepping the viewport 320→1920px in 8px increments on both
routes, reporting element overflow, text overlap and document horizontal scroll. Detector
self-tested against injected faults (0 false positives on a clean page). Every width and
measurement below is measured, not estimated.

**The short version:** this site is in good shape. Type uses `clamp()` throughout,
`prefers-reduced-motion` is covered in three separate blocks, `.final` uses `svh` and has a
dedicated short-landscape rule, and `section[id]` already carries `scroll-margin-top` for the
sticky nav. **Everything from 345px up is clean** — no overflow, no overlap, at any width to
1920px, in both pointer modes. There is one real breakage, confined to viewports ≤344px.

---

## Rules

Standing rules for this site. Everything below is audited against them.

1. **1512px is the primary desktop width** (MacBook Pro 14"). 1920 is spot-check only.
2. **Use `dvh`, not `vh`,** for full-height sections.
3. **Add `env(safe-area-inset-*)` padding** wherever content reaches a screen edge.
4. **All inputs need `font-size: 16px` minimum** (below that, iOS zooms the page on focus).
5. **Test iPhone landscape at 852×393.**
6. **No layout jumps, overlapping text, or horizontal scroll anywhere between 320px and 1920px.**

Standard width matrix (`shots.mjs` default), real devices rather than round numbers:

| 375 | 393 | 430 | 834 | 1024 | 1512 | 1728 | 1920 |
|---|---|---|---|---|---|---|---|
| iPhone SE / 13 mini | iPhone 14/15 | iPhone Pro Max | iPad portrait | iPad landscape | **MBP 14" (primary)** | MBP 16" | desktop |

### Rule compliance

| Rule | Status |
|---|---|
| 1 · 1512 primary | ✅ **Pass** — nothing breaks; nav and content wraps both 1120px with identical left edges at 1120/1280/1512/1728/1920; `scrollWidth` == viewport |
| 2 · `dvh` not `vh` | ❌ **2 violations** — see R2a, R2b |
| 3 · safe-area insets | ❌ **1 violation, site-wide** — see R3 |
| 4 · inputs ≥16px | ✅ **Pass** — see R4 note |
| 5 · 852×393 landscape | ✅ **Pass** — 0 overflowing elements, `scrollWidth` 852, `.final` 383px inside a 393px viewport |
| 6 · nothing broken 320→1920 | ✅ **Pass** — horizontal scroll and overlap both clear after fixing item 1; all six height discontinuities accounted for, see R6 |

---

## Rule violations

- [x] **R2a · `.final` uses `svh`/`vh`, not `dvh`** — `index.html:207`.
  `min-height:70vh;min-height:70svh`. Rule 2 asks for `dvh`.
  **Worth a decision rather than a blind fix.** The `vh` line is a deliberate fallback for
  browsers without viewport variants, and the comment above it explains the `svh` choice.
  `svh` is the *small* viewport height (browser chrome expanded) — it can never overflow, but
  leaves a gap when the chrome retracts. `dvh` tracks the chrome live, which fills the screen
  but reflows the section while the user scrolls. For a centred full-screen closing panel,
  `svh` is arguably the safer of the two. If the rule is absolute, change to
  `min-height:70dvh` and keep `70vh` as the fallback.
  **All widths, phones only** (desktop chrome doesn't retract).

  **Fixed** — now `min-height:70vh;min-height:70dvh`. The rule was applied over the author's
  `svh` preference; the tradeoff is written into the code comment rather than dropped, since
  only 70% of the viewport is claimed and neither behaviour is dramatic at that size.
  Measured unchanged in headless Chromium (596px at 393×852, 467px at 375×667, and the
  `max-height:560px` rule still zeroes it at 852×393) — see the R2b caveat on why the real
  difference is not observable here.

  **Two `vh` uses deliberately left alone** — `.final .row` `margin-top:clamp(34px,5vh,58px)`
  (line 254) and the short-viewport `padding:clamp(56px,10vh,90px)` (line 257). Both are
  spacing inside tight clamps, not full-height section sizing, so rule 2 does not reach them;
  converting would change the result by a few pixels at most.

- [x] **R2b · `.rs-scr` sizes itself from raw `100vh`** — `index.html:397`.
  `height:max(430px, min(604px, calc(100vh - var(--nav-h,64px) - 96px)))`. This is a
  full-height-derived measurement and `100vh` is the retracted-chrome height on a phone, so
  the phone mockup is computed taller than the space actually visible. Impact is bounded by
  the `min(604px, …)` cap and the `max(430px, …)` floor — measured 604px at 393×852 and 430px
  at 852×393, both sane — so this is latent rather than currently visible. `100dvh` is the
  correct unit here.
  **Phones, portrait.**

  **Fixed** — `100dvh` added as a second declaration, keeping the `100vh` line above it as the
  fallback for browsers without viewport variants. Measured after the change: 604px at
  393×852 (the `min()` cap), **514px at 375×667** (the calc branch: 667 − 57 nav − 96), 430px
  at 852×393 (the `max()` floor). All three branches still behave.

  **Caveat on the verification:** headless Chromium has no retracting browser chrome, so
  `vh`, `svh` and `dvh` all resolve to `innerHeight` there. What is proven is that the
  declaration parses, wins, and leaves every branch sane; the actual behavioural difference
  only appears on a real phone.

- [x] **R3 · No safe-area handling anywhere on the site** — `index.html:5, 87, 245`.
  `env(safe-area-inset-*)` appears **0 times**, and the viewport meta is
  `width=device-width, initial-scale=1` with no `viewport-fit=cover`.
  Content reaching a screen edge: the sticky `header.nav` (top), `footer` (bottom), and
  `.wrap`'s 26/18/15px gutters (left/right). On a notched iPhone in landscape the gutter falls
  under the notch, and the footer runs under the home indicator.
  **Note the two must ship together** — adding `viewport-fit=cover` alone extends layout *into*
  the unsafe area and makes it worse. Use `max(26px, env(safe-area-inset-left))` so the design
  gutter is preserved and only grows where a notch actually intrudes.
  **≤926px landscape on notched devices.**

  **Fixed** — `viewport-fit=cover` added to the meta, plus `max(gutter, env(...))` on `.wrap`
  and both its responsive overrides (600px → 18px, 360px → 15px, which used the `padding`
  shorthand and would otherwise have wiped the insets), `padding-top` on `header.nav`, and
  `env(safe-area-inset-bottom)` on the footer.

  **No JS change was needed, which is worth recording.** `--nav-h` is published from
  `bar.offsetHeight` (`index.html:845`), and `offsetHeight` includes padding — so the nav's new
  top inset propagates automatically to all four consumers (`.rs-stack` sticky offset,
  `.rs-scr` height, `section[id]` scroll-margin-top, `.final` padding). Verified by simulating
  a 59px inset before load: `--nav-h` goes 65→**124px** at 852×393 and 57→**116px** at 393×852,
  and `scroll-margin-top` follows to 138/130px. Desktop is untouched (`env()` → 0, gutters stay
  26/18/15px). 0 overflowing elements at both orientations.

  One residual fragility, not currently reachable: the `ResizeObserver` watching the nav uses
  the default content-box, so a padding-only change does not fire it. Insets change on
  rotation, which fires `window.resize` and re-runs `measure()`, so the real path is covered —
  but a future padding change with no resize would leave `--nav-h` stale.

- [x] **R6 · Layout jumps at the 840 and 900 breakpoints** — measured height deltas of
  **−872 to −894px at 848px** and **−1068 to −1088px at 904–912px**.
  Both are the intended breakpoints reflowing (`.hero`/`.feat` → 1 column at 840; `.rs` → 1
  column at 900), not accidents.
  **848px and 904–912px.**

  **Resolved — no code change, deliberately.** Rule 6 has three parts. Two are absolute and now
  pass; the third is not achievable as literally written.

  - **No horizontal scroll 320→1920** ✅ — was violated ≤344px, fixed (item 1).
  - **No overlapping text 320→1920** ✅ — 0 overlaps at 4px resolution.
  - **No layout jumps** — re-swept at **4px steps with the threshold dropped to 40px**, which
    surfaces six discontinuities rather than two. Every one is accounted for:

    | Width | Δheight | Cause |
    |---|---|---|
    | 344px | −74px | `.lm-bar`'s two buttons stop wrapping to a second row (−52px) + a `p.body` line |
    | 604px | +111px | the **600px** breakpoint |
    | 720px | −48px | hero `h1` goes 2 lines → 1 (108→54px) |
    | 764px | +74px | the **760px** breakpoint |
    | 844px | −904px | the **840px** breakpoint (`.hero`/`.feat` → 1 col) |
    | 904px | −1099px | the **900px** breakpoint (`.rs` → 1 col) |

    Four are declared breakpoints; two are text rewrap. Neither category is removable: a
    two-column→one-column transition changes height by definition, and moving to
    `minmax()`/`auto-fit` relocates the jump rather than removing it. Text rewrap changes
    height at every width where a line breaks, in any layout.

  **There are no unintended structural jumps.** If the rule is meant literally — genuinely
  continuous height across the whole range — it would require abandoning the two-column
  design, which is a redesign rather than a fix. Recommend reading it as "no *unintended*
  jumps", which is now satisfied and enforced by the sweep.

### R4 note — inputs (no action needed)

Rule 4 passes, with one thing worth recording so it isn't re-flagged later. Measured at 393px:

| Control | font-size |
|---|---|
| `textarea.ir-field` | **16px** ✅ |
| `select` (typeface picker) | **16px** ✅ |
| 4 × `input[type=range]` | 13.33px (UA default) |

The range inputs are below 16px but render no text and cannot receive a text caret, so they
can't trigger iOS's zoom-on-focus — which is the behaviour rule 4 exists to prevent. Both
controls that *can* take focus for text entry are already at exactly 16px.

---

## Breakpoint inventory

| Width | Line | What it owns |
|---|---|---|
| `≤900` | 547 | `.rs` → 1 col, `.rs-stack` unsticks, phone centred |
| `≤840` | 249 | hero/feat → 1 col, demo to top, **`.nav-links` hidden** |
| `≤760` | 308 | `.hero.solo` type scale |
| `≤760` | 329 | `.sh` → 1 col — **dead, inside an unterminated comment (see 5)** |
| `≤700` | 577, 614 | `.lm-heap` height 520→580px, `.lm-stage`/`.lm-note` padding |
| `≤600` | 254 | phone tier: `.wrap` padding, nav height, **`.theme-dots` hidden**, type scale |
| `≤360` | 276 | small-phone tier: `.wrap` padding 18→15px, `h1` 32px |
| `≤560` height | 244 | `.final` drops `min-height` for landscape |
| reduced-motion | 54, 69, 236 | animation/transition/scroll-behaviour off |

---

## Findings

All open items, most severe first. Details for each follow below.

- [x] **1 · Reshaper overflows, cut content unreachable** — ≤344px. *Also the rule 6 violation.* **Fixed.**
- [x] **R3 · No safe-area insets anywhere** — ≤926px landscape, notched devices. **Fixed.**
- [x] **2 · `privacy.html` ignores colour scheme** — all widths. *Shared with the iOS bundle.* **Fixed.**
- [x] **3 · Touch targets below 44px** — 19 of them; range sliders are 16px tall. **Fixed — 0 remain.**
- [x] **R2b · `.rs-scr` sized from raw `100vh`** — phones, portrait. Latent. **Fixed.**
- [x] **R2a · `.final` uses `svh`/`vh` not `dvh`** — phones. **Fixed** (rule applied over the
  author's `svh` preference; tradeoff recorded in the code comment).
- [x] **4 · Section nav vanishes ≤840px with no replacement** — **Partly fixed:** recovered
  761–840px; still hidden ≤760px, where it needs a design decision.
- [x] **R6 · Layout jumps at 848px and 904–912px** — **Resolved, no code change.** All
  discontinuities accounted for; the two hard parts of rule 6 now pass.
- [x] **5 · Unterminated CSS comment** — latent trap, no current effect. **Fixed.**
- [ ] **6 · `body{overflow-x:hidden}` masks overflow** — all widths; makes item 1 unreachable.

### 1. The Reshaper section overflows and the cut content is unreachable — **≤344px**

`.rs-phone` is a hard `width:330px` (line 390). Below 900px `.rs` collapses to a single
column, so that fixed width becomes the column's min-content and **every sibling is stretched
to 330px**: the option trays, the Coloured toggle, the typeface row and both sliders. At 320px
`.wrap` gives a 290px content box, so the whole block sits 25px past the viewport edge.

Measured at 320px: `.rs-controls`, `.rs-tray`, `.rs-textrow`, `.rs-slider`, `.cz-toggle` and
`.rs-phone` all end at **x=345** against a 320px viewport; `document.scrollWidth` is 349px.

Visibly cut: **Shelf → "Shel"**, **Solid → "Soli"**, **Ledger → "Ledge"**, **Bar**,
**Forest → "Fores"**, the Coloured toggle severed mid-switch, both slider values ("15 pt",
"9 pt") off-screen, and the phone mockup's right edge past the fold.

**This is worse than a normal overflow:** `body{overflow-x:hidden}` (line 51) suppresses
horizontal scrolling, so `scrollLeft` is capped at **0**. The user cannot pan to reach the cut
options — on a 320px phone, five of the app's options are simply not selectable.

Affects iPhone SE 1st/2nd gen and any 320px device. Clean from 345px up. **This is the
rule 6 violation** (horizontal scroll between 320 and 1920).

**Fixed** — two changes, because the first alone made it worse:

1. `.rs-phone` → `width:min(330px,100%)`, so it stops being a hard floor.
2. `.rs > * { min-width: 0 }` — the actual cause. Grid items default to
   `min-width:auto`, so a column will not shrink below its content's min-content width.
   Measured at 320px: `.rs-scr`'s interior has a **405px** min-content, which propagated up
   (`.rs-phone` 429px → `.rs-stack` 429px) and pinned the single column at 429px inside a
   290px box. Change 1 on its own removed the 330px cap and let that 429px floor take over
   instead, pushing the overflow from ≤344px out to ≤400px and `scrollWidth` from 349 to
   451px. Only `min-width:0` releases it.

The interior already clips (`.rs-scr` is `overflow:hidden`), so the phone now scales 330→290px
at 320px viewport: the chip row clips "Reading" and the notebook title wraps, but everything
is on-screen and reachable. Verified: sweep clean **320→420px at 4px resolution** and
**320→1920px at 8px** on both routes; all option labels, the toggle and both slider values
visible at 320px.

### 2. `privacy.html` ignores the viewer's colour scheme — **all widths**

Colours are hard-coded (`#F2EDE1` paper, `#2A2118` ink) with no `prefers-color-scheme` block.
Measured: background and ink are byte-identical in light and dark. The main page re-themes
across five palettes, so following the footer's Privacy link in a dark theme flashes
full-brightness.

Note this file is **byte-identical to `ios/Lembaran.swiftpm/privacy.html`** in the Lembaran
repo, which ships inside the app bundle. Fix both together or they drift.

**Fixed** — colours moved into five custom properties with a `prefers-color-scheme: dark`
block using the app's `midnight` palette. Verified at all 8 matrix widths in both schemes:
light resolves to **exactly the original values** (`rgb(242,237,225)` / `rgb(42,33,24)` /
`rgb(158,59,46)`), dark to `rgb(32,28,23)` / `rgb(234,225,208)` / `rgb(210,142,82)`, with no
horizontal scroll at any width.

⚠ **The iOS copy is now out of sync.** `ios/Lembaran.swiftpm/privacy.html` in the Lembaran repo
still has the hard-coded colours on `main`. An equivalent fix exists there on the unmerged
`responsive-fixes` branch (commits `e7d367b`, `0b0e060`), but that version *also* scales the
margins and headings, so the two files will not be byte-identical until one is reconciled to
the other.

### 3. Touch targets below 44px — **coarse pointer, all widths**

19 interactive elements measure under 44px in their smaller dimension at 375px. The ones that
matter most:

| Element | Size |
|---|---|
| header `a.btn` ("Get the app") | 109×35 |
| `button.toggle` (Coloured) | 48×28 |
| typeface `<select>` | 101×18 |
| `input[type=range]` (font size / line spacing) | 339×22 and 339×16 |
| `.pay-opts button` | 84×30 |
| footer links | 37×16, 44×16, 106×16 |

The range inputs are the sharpest: a 16px-tall drag target is hard to grab on a phone, and
they are the primary control in the "Note text" group.

**Fixed — 19 → 0** under an `@media (pointer: coarse)` block; fine pointers keep the tighter
design unchanged. Full inventory was 11 distinct kinds, not the 6 first sampled — `.vn-controls`
range, `.pro .row2` links and the footer links were only found by enumerating rather than
spot-checking.

**`.toggle` took three attempts, and the first two were wrong:**

1. A `::before` with negative insets to expand the hit area invisibly. **Did not work** —
   hit-testing 6px above and below the pill still missed it. Discarded rather than debugged.
2. Scaling the whole pill to 72×44. Passed the target check but **broke the layout**: +24px on
   `.cz-toggle`'s min-content propagated up and pushed the document to 325px at a 320px
   viewport. Caught by re-sweeping, then isolated by disabling each coarse rule in turn.
3. Height only, 28→44px, with the knob re-centred (`top:11px`). The pill is already 48px wide,
   so only the short dimension needed to move. No horizontal impact.

Verified: **0 targets under 44px** at 393px, 0 overflowing elements at 320px, touch and fine
sweeps both clean 320→1920, and screenshots at all 8 matrix widths confirm desktop is
visually unchanged.

### 4. Section navigation disappears below 840px with no replacement — **≤840px**

`.nav-links{display:none}` (line 252) drops all five section links (Write / Organize / Private
/ Yours / Price) and nothing takes their place — no hamburger, no in-page index. On a long
single-scroll page this is defensible, but it is a silent loss of the only wayfinding.

Worth contrasting with `.theme-dots{display:none}` at ≤600px (line 258), whose comment claims
"full theme picker lives in Make it yours". **That claim is true and verified** — clicking a
theme option inside `#yours` at 375px changes `data-app-theme` and re-themes the page. That
one is a sound trade; the nav links are not covered the same way.

**Partly fixed — breakpoint moved 840 → 760.** The 840px threshold was far more conservative
than needed. Measured: the five links are **294px** and fit alongside the brand, theme dots and
CTA down to **760px** — 46px of slack at 840, 8px at 760. So the whole **761–840px tablet band**
was hiding them for no reason. Verified after the change: links render from 764px up with the
last link ending at x=465 against a CTA starting at x=649 (184px clear) at every width to 1920.
iPad portrait (834px) now shows the full nav; phones are unchanged.

**Still open below 760px, and it needs your call.** Below 760 the links genuinely collide —
brand + links + theme dots + CTA exceeds the bar. The options, none of which I applied
unilaterally:

- **Hide the theme dots earlier** (≤760 instead of ≤600). The links would then survive to
  ~560px. Defensible, since `#yours` already carries a verified full theme picker — but it
  removes a control that 600–760px users currently have.
- **Add a menu button.** Proper wayfinding at every width, but it is a new component with JS,
  focus management and ARIA — a feature, not a responsive fix.
- **Leave it.** The page is a single linear scroll; the links are a convenience, not the only
  route to any content.

### 5. Unterminated CSS comment silently swallows rules — **latent, no current effect**

Line 323 opens a comment that is never closed:

```css
/* A note dropping into the middle of the pile{animation:sh-in .42s cubic-bezier(.2,.8,.2,1) both;}
```

The `*/` is missing and a selector has been absorbed into the comment text. Everything up to
the next `*/` — which is 12 lines later, at the end of the `#private` comment block — is
therefore commented out, including `@keyframes sh-in` and
`@media (max-width:760px){.sh{grid-template-columns:1fr;}}`.

**No current rendering impact:** `.sh` markup no longer exists (the `#organize` section was
rebuilt around the `.lm` tidy-up demo), so everything swallowed is already dead. Confirmed the
comment closes where expected — `.ir-field` immediately after it computes its `14px` radius,
so live CSS resumes correctly.

It is still worth fixing, because it is a trap: any rule added between line 323 and line 335
will silently do nothing.

**Fixed** — the whole orphaned block is gone rather than just the comment closed: the broken
comment, `@keyframes sh-in`, the `.sh` rule and its `≤760px` media query, none of which had
markup any more. The stale section comment above them (which described the `.sh` side-by-side
comparison) was rewritten to describe what `.feat.solo` actually does now.

Verified the removal was inert by comparing **computed styles for every element across 34
properties, all 8 matrix widths and both routes, with animations frozen** — byte-identical
before and after. (Screenshot hashes are not usable for this: the page runs continuous
animations, so two runs never hash alike.)

### 6. `body{overflow-x:hidden}` masks overflow — **all widths**

Line 51. On `body` this propagates to the viewport and suppresses horizontal scrolling
site-wide. It doesn't cause finding 1, but it converts it from "content off the right edge you
can pan to" into "content that cannot be reached at all". It also means overflow bugs won't
announce themselves during manual resizing.

---

## Checked and found sound

Listed so absence from the findings means examined-and-fine, not unexamined.

- **All widths 345→1920px** — no overflow, no overlap, on both routes, in both fine and
  coarse pointer modes.
- **`privacy.html` layout** — clean 320→1920px at 8px resolution. Only its colour scheme is a
  problem (finding 2).
- **Landscape / short viewports** — 0 overflowing elements at **852×393 (rule 5)**, 844×390 and
  740×360. The `@media (max-height:560px)` rule correctly drops `.final`'s `min-height`
  (measured 383px inside a 393px viewport, rather than a full screen), and `.rs-scr`'s
  `max(430px, …)` floor holds at 430px. This is handled better than most sites.
- **1512px, the primary desktop width (rule 1)** — `header.nav .wrap` and every content
  `.wrap` are both 1120px with identical left edges (196px) — measured at 1120/1280/1512/1728/
  1920, differing by 0px at every one. `.hero.solo .wrap` is deliberately full-width but its
  `h1` and lede are `ch`-constrained and centred, so nothing runs wide. `scrollWidth` equals
  the viewport.
- **Text-entry input sizing (rule 4)** — both focusable text controls are exactly 16px.
- **Theme picker replacement** — verified functional at 375px, so hiding `.theme-dots` is safe.
- **Anchor offsets** — `section[id]{scroll-margin-top:calc(var(--nav-h,64px) + 14px)}` already
  keeps targets clear of the sticky nav.
- **Type scaling** — `clamp()` on every heading and lede, with explicit small-phone overrides
  at 600px and 360px.
- **Reduced motion** — three blocks covering animations, transitions, scroll-behaviour, the
  reveal/enter states and the final CTA's hover transform.
- **`.lm` tidy-up demo** — fixed stage heights (612px ≤700px, 564px above) but no overflow at
  any width; the heap's `max-width` correctly switches 42%→76% at the 700px breakpoint.

---

## Reproducing

Tooling was kept outside this repo to preserve its zero-build, zero-dependency setup — it
lives in the session scratchpad. If you want it committed here permanently, say so and it can
be added along with a `package.json` (it needs only `playwright-core`).

```
node shots.mjs                 # the 8-width device matrix above
node shots.mjs --el "#yours"   # one section, same widths
node sweep.mjs                 # 320→1920, 8px steps, both routes (rule 6)
node sweep.mjs --touch         # coarse-pointer pass
node sweep.mjs --selftest      # prove the detector still fires
node rules.mjs                 # rules 1, 4 and 5
```

`shots.mjs` emulates a coarse pointer at ≤1024px, so the iPad widths exercise touch rules too.

Do not compare screenshot hashes to prove a CSS change is inert — this page runs continuous
animations, so two runs never hash alike. Compare computed styles with animations frozen.
