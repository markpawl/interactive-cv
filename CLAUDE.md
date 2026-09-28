# CLAUDE.md — Mark Pawlowski resume site

Guidance for Claude Code when working in this repo. It captures the decisions, behavior rules and working
preferences that were established while the site was designed in a chat session. Treat the behavior rules
below as the spec: do not change them unless the user asks.

## 1. What this is

A single-page, static résumé site with two tabs:

- **Job Specific Qualifications** (main tab): sections of intro text + bullets. Titles/bullets can carry extra
  "hover" text that appears in a Details box when clicked. All of this text comes from `resume-content.json`.
- **Traditional CV**: a hand-written HTML rendering of the PDF CV, plus a link to download the PDF.
  This text is **not** in the JSON; it lives in `index.html` (`#cv-view`).

Hosted as a static site on Vercel. No framework, no build step, no dependencies, no external fonts/scripts/CDNs.

## 2. Files

| File | Purpose |
|---|---|
| `index.html` | Entire app: HTML + CSS + JS in one file. Builds the main tab from the JSON at load. |
| `resume-content.json` | **All text of the main tab** (header, tab labels, messages, sections). The file the user edits most. |
| `mp-traditional-cv.pdf` | Downloadable CV, linked from the Traditional CV tab (relative link, must sit next to `index.html`). |
| `CLAUDE.md` | This file. |

`resume-source.md` (an older markdown source) is **retired**. The JSON replaced it; do not regenerate from it.

## 3. Run and test locally

`index.html` fetches `resume-content.json` at runtime, and browsers block that on `file://`. Serve the folder:

```bash
python3 -m http.server 8000      # then open http://localhost:8000
```

Opened straight from disk, the page shows an on-screen banner explaining this. That is expected.

Deployment (Vercel): framework preset "Other", no build command, no output directory; the repo root is the site.

Dormant hook: if the page contains `<script type="application/json" id="resume-data">…</script>`, `readContentText()`
uses it instead of fetching. This only exists so a single-file preview can be built by inlining the JSON; it does
nothing on the real site. Do not add an embedded copy of the content to `index.html`.

## 4. `resume-content.json`

Top-level keys: `_readme` (notes, ignored by code), `header`, `tabs`, `labels`, `sectionOrder`, `sections`.

```jsonc
"header":  { "pageTitle", "name", "tagline", "contact": ["{markpaw@gmail.com|mailto:…}", "phone", "city"], "status" },
"tabs":    { "main": "Job Specific Qualifications", "cv": "Traditional CV" },
"labels":  { "navHeading", "hoverPrompt", "emptyMessageDesktop", "emptyMessageTouch", "noInfoMessage" },
"sectionOrder": ["documentation", "ai-tools", ...],           // order on the page AND in the Background list
"sections": {
  "some-id": { "type": "standard", "title", "hover", "intro",
               "bullets": [ { "text", "hover" } ] },
  "links-id": { "type": "links", "title", "hover", "intro",
                "bullets": [ { "text", "url" } ] }            // plain links, opens in a new tab, not selectable
}
```

Rules:
- **Order**: reorder by moving lines in `sectionOrder`. A section defined but not listed is hidden (console.info only).
  A listed id that does not exist is a fatal error.
- **Explicit "no hover"**: `hover` is `"na"` or `""` when there is none (case-insensitive, whitespace ignored; a missing key
  is treated the same). Keep writing the explicit value in the data; the user wants the lack of hover text to be visible.
- **Rich text** (via `renderRich()`, used for intros, bullets, hover text, tagline, status, contact items):
  `**bold**`, `\n` for a line break, `{label|url}` for links (`https?:`, `mailto:`, `tel:`; external http(s) links open in a new tab).
  Everything else is HTML-escaped. Never insert JSON-derived text with `innerHTML` unless it went through `renderRich()`
  or `escapeHtml()`. Titles and labels are plain text (`textContent`).
- **DOM ids** for sections are `"sec-" + key`. Keys should be lowercase-hyphenated.
- **Errors** show a red banner (`#data-error`) and hide the main layout instead of failing silently: missing file (HTTP status),
  JSON syntax error (with line number), missing title/bullets/url, bad `sectionOrder`. Keep messages specific and actionable.
- The fetch uses `cache: "no-cache"` so edits show up on a normal reload.
- User wording is kept **verbatim**. Do not "improve" their text; list suspected typos/grammar separately and ask first.
  (Typos and four grammar items were already fixed with approval.)

## 5. Behavior spec: main tab

### Selecting items and the Details box
- Every section title and bullet is selectable (pointer cursor, hover tint), except bullets in `links` sections.
- **Hover** (mouse only, and only when nothing is selected): items *with* hover text show **"Click for related information!"**
  centered, in the link blue, in the Details box. Items with `na`/`""` hover show nothing on hover.
- **Click**: selects the item. Selected style = light-blue background + left accent bar on the item, and the Details box turns
  light blue with the hover text (links inside it are live). An item with no hover text, when clicked, shows
  **"No extra information for this item."** (neutral gray box).
- While an item is selected, hover messages are suppressed everywhere (the selected item's text must stay visible).
- **Deselect** happens when: another item is clicked; the selected item is clicked again (toggle); a click lands elsewhere in the
  document that is not a selectable item; a click lands outside the document (header, rail, tabs); or the selected item scrolls
  out of view (IntersectionObserver on the document scroller).
- The Details box shows **only** the text/messages: no title, no "Detail"/"Section" label.
- A link inside a bullet's text follows the link and does not select the bullet.
- Selected styling must win over hover styling (mind CSS order/specificity; a past bug turned the selected item gray on hover).

### Background list (nav)
- Lists section titles. Clicking one smooth-scrolls the document scroller so that section's title sits at the top.
- Highlights the section currently at the top of the scroller (blue text + light-blue background).
- Desktop: always shows its full contents (no scrolling, no fixed 50% height). The Details box sits **directly beneath** it and is only as tall
  as its text (scrolls internally if it can't fit).

### Header
- The name is an `<h1>` containing a toggle `<button>`: a triangle inside a **borderless circle** (light gray background) before the name.
  Points **right** when collapsed, **down** when expanded (aria-expanded drives the rotation).
- The collapsible section holds, in order: tagline, contact row (items separated by `·`), status line. It is **open by default**.
- **Auto-close 10 s after page load** (`BIO_AUTO_CLOSE_MS`). Any manual click on the toggle cancels the timer permanently.
- After an *auto*-close only, the circle's background **blinks for 3 s** (5 × 0.6 s) to hint that it can be reopened. Manual closes do not blink.
  `prefers-reduced-motion` disables the blink (the user may not see it because of that setting; see follow-ups).
- LinkedIn/GitHub are intentionally **not** in the header; they live in the Websites section.
- Header tooltip was removed; the tagline is shown in the collapsible section instead.

### Tabs
- Tab bar sits on the bottom edge of the header: **Job Specific Qualifications** and **Traditional CV**. Folder-style tabs:
  unselected = light blue (`--tab-bg`) with blue-tinted border; selected = page background, near-black **bold** text, and a muted
  steel-blue (`--tab-accent`) bar across its top edge. The selected tab covers the full-width baseline so it merges into the page.
  The tabs deliberately do **not** use the link blue, so "selected tab" and "selected item" are not confused.
- Tab text is small on purpose (11.6 px desktop, 10.8 px phone). Bold width is reserved via `data-text` + `::after` so tabs never resize on select.
- The CV tab lives at hash **`#cv`**. Browser/phone Back and Forward switch tabs; opening `…/#cv` directly shows the CV.
  Switching back to the main tab restores its scroll position. Selection is cleared when switching to the CV.
- Arrow Left/Right switch tabs when a tab is focused.

### Traditional CV tab
- Replaces the main layout; the site header (name, collapsible section, tabs) stays. The CV has **no header of its own**
  (no name/contact block; no street address in the HTML, though the PDF has one).
- Directly under the tab bar: a full-width light-gray band with the link **"click here to download pdf copy"**
  (12 px, dark gray `--band-text`, no underline at rest, underline on hover, **left-aligned**, opens the PDF in a new tab).
  No extra padding above/below the band.
- CV content is **left-justified** (max-width 820 px), aligned with the header's left edge (48 px desktop, 16 px phone).
- Skills are pipe-separated lists that only wrap **between** items (`inline-block; white-space: nowrap` items with a `|` separator).
- Contents: Courseware Production & Training Skills, Work Experience, Achievements, Education, Language Skills, Outside Interests.
  The citizenship/visa line ("US citizen - Holding a valid German Work Visa.") lives in the header status line, without "Able to join within 2 weeks".
- The HTML CV and the PDF are two separate copies. When the CV changes, update both.

## 6. Layout and responsive rules

- **Desktop / any width ≥ 601 px**: header on top; below it a two-column area. Left **70 %** = the scrolling document. Right **30 %** = fixed rail
  (Background list, then Details box). **The right panel must always be visible on desktop.** The `@media (min-width: 601px)` block forces
  this with `!important`; do not lower or remove it.
- **Phone (≤ 600 px)**, app-like rather than website-like:
  - Compact header. Background list becomes a **single row** of pill buttons, horizontally scrollable, at the top; the active pill auto-centers.
  - The document scrolls in the middle.
  - The Details box is a **footer fixed to the bottom of the viewport** (`position: fixed`), always visible (not at the end of the document),
    sized to its text, max `35vh`, scrolls internally beyond that, respects `env(safe-area-inset-bottom)`.
    JS publishes its height as `--footer-h` so the document leaves room; when the footer grows over the selected item, the document is nudged so the
    item stays visible (otherwise the item would count as "scrolled out of view" and be deselected).
  - No hover on touch: the hover prompt is skipped (`(hover: hover)` check); the empty message uses the touch wording ("Tap…").
  - `aside.side-rail { display: contents }` on phones so the nav and footer can be ordered independently of the document.
- Vertical spacing is deliberately **tight** throughout (document, nav list, details area). Do not loosen it without being asked.
- No horizontal page scroll at any width (`document.documentElement.scrollWidth <= innerWidth`).

## 7. Visual style

Matches the look of an Apple Support article: SF Pro system font stack, white/#f5f5f7 backgrounds, thin `#e6e6e6` rules, generous restraint.
Colors are CSS variables on `:root` with a `prefers-color-scheme: dark` override block; add new colors as variables, in both themes.

| Variable | Light value | Use |
|---|---|---|
| `--base-color` | `#1d1d1f` | text |
| `--emphasis-color` / `--footer-color` | `#555` / `#86868b` | secondary text / labels |
| `--border-color` / `--panel-border` | `#e6e6e6` / `#d2d2d7` | rules |
| `--link-color`, `--highlight-border` | `#007aff` | links, selection accent |
| `--highlight-bg` | `rgba(0,122,255,.07)` | selected item / active nav |
| `--panel-bg` | `#f5f5f7` | details box (idle), PDF-link band |
| `--tab-bg`, `--tab-border`, `--tab-accent` | `#e4eefa`, `#c5d9f0`, `#6f9ac6` | tabs |
| `--band-text` | `#48484a` | PDF link text |
| `--tri-bg`, `--tri-blink` | translucent gray / `rgba(0,122,255,.55)` | header toggle circle / blink color |

## 8. Code notes and gotchas

- Main script is an `async` IIFE: `loadContent()` → build header → build sections → wire interactions. Key functions:
  `renderRich`, `isNa`, `attachInteractions`, `selectItem`/`clearSelection`, `showHoverMessage`, `renderSelectedInfo`, `syncFooter`,
  `syncTabs`/`showTab`, `setBio`. State: `selectedEl`, `selectedData`, `hoverActive`.
- Don't put `overflow-x: auto` on an element that relies on child negative margins to overlap its border (it clips). The tab bar therefore draws its
  own baseline with `::after` and stacks tabs above it with z-index.
- Touch devices: `@media (hover: none)` neutralizes sticky hover backgrounds after a tap.
- The document scroller (`#doc-scroll`) is the scroll container on all sizes; scroll/IntersectionObserver logic is anchored to it.
- Browser storage APIs are not used. Keep it that way unless needed.
- Keep it dependency-free and offline-capable (system fonts only).

## 9. Regression checklist (run after any change)

Test at **1280×800** and at **390×844 touch/mobile** (Playwright + Chromium worked well; serve over http, not `file://`):

1. Page renders all sections in `sectionOrder`, no red banner, no console errors, no horizontal scroll.
2. Nav click scrolls the section to the top; active nav item follows scrolling.
3. Selection rules in §5 (hover prompt, click, "no extra information", toggle off, click-away, scroll-out, links inside text).
4. Header toggle: default open, triangle direction, auto-close at 10 s, manual click cancels it, blink only after auto-close.
5. Tabs: switch by click, Back/Forward, `…/#cv` deep link, scroll restored on return.
6. Phone: footer's bottom edge equals the viewport height at every scroll position; selected item never hides behind it.
7. Error paths: missing JSON (404), syntax error (line number shown), missing `title`, unknown id in `sectionOrder`, `links` bullet without `url`.
8. Dark mode looks right (not yet reviewed for tabs and the header circle).

## 10. How to work with this user

- **"Discuss first" means stop.** When the user says to talk something through or asks what you think before executing, give options, a recommendation and any
  questions, then wait. Do not implement until they agree.
- Make **exactly the change requested**, no redesigns or extras. Show/describe what changed and any trade-off in a few sentences.
- If a request is ambiguous or self-contradictory, pick the most plausible reading, **say which one you chose**, and offer the alternative.
  (Example: "instead of right-justified, right-justify" was implemented as left-justify and flagged.)
- Never silently reword the user's text. Report suspected typos/grammar issues in a separate list and ask before changing them.
- They test on both desktop and phone and care about the phone feeling like an app.
- Prefer suggestions over unilateral choices for subjective visual issues (e.g. color confusion): offer 3–4 options with a recommendation.
- Keep replies short and plain; list files touched when relevant.

## 11. Known follow-ups / open items

- The header blink was "not noticeable" to the user, likely because of a reduced-motion OS setting. Idea: a steady highlight (or stronger cue) for
  reduced-motion users.
- `resume-content.json` `_readme` still says "block" and warns about a closing script tag; both are leftovers from when the JSON was embedded in the HTML. Clean up.
- Phone tab labels are 10.8 px; may want to increase.
- Dark mode not visually reviewed for the tabs, PDF band and header circle.
- CV text could move into the JSON later (user said not needed now).
- The single-file preview build used during chat design (JSON inlined) is not part of this repo.
