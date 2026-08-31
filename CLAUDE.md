# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-page training/demo mockup of an internal IT Support Service Desk, branded "UOB Bank"
for classroom use. The entire project is one file: `index.html` (~1,600 lines) containing the
markup, a `<style>` block, and a `<script>` block. The visual direction is a dark-green /
terminal "tech" theme; the animation is all CSS keyframes plus one IntersectionObserver.

## The hard constraints

These are the point of the exercise, not incidental. Breaking any of them breaks the deliverable:

- **One self-contained file.** No frameworks, no build step, no package manager, no external
  requests. It must work by double-clicking from `file://`.
- **No persistence, no network.** `localStorage`, `sessionStorage`, `fetch`, and
  `XMLHttpRequest` are forbidden. Submitted tickets are pushed onto the in-memory `tickets`
  array and are intentionally lost on reload.
- **No external assets.** The only SVGs are inline; the select chevron is a `data:` URI. The
  sole allowed `http` string in the file is the SVG namespace `www.w3.org/2000/svg`. This rules
  out Google Fonts — `--font` / `--font-mono` are stacks that name real families first and fall
  back to platform fonts, never an `@import`.
- **The CSP meta tag stays.** `<head>` carries a `default-src 'none'` Content Security Policy.
  It is what actually makes "no network" enforceable rather than aspirational, and the Security
  section of the page describes it to the reader — if you change one, change both.
- **Never collect credentials.** No password/OTP/PIN/card fields, and the description field
  actively warns when it looks like the user pasted one.
- **The disclaimer stays visible.** "Training demo — not affiliated with or endorsed by United
  Overseas Bank Limited" must remain in the footer bottom bar.

## Commands

```bash
# Open it (this is the whole "dev loop" — edit, save, reload the browser)
start index.html                       # Windows
```

There is no test framework. Verification is ad-hoc Node against the source:

```bash
# Constraint check — must print only the comment line that mentions them
grep -inE "localStorage|sessionStorage|fetch\(|XMLHttpRequest|https?://" index.html \
  | grep -v "www.w3.org/2000/svg"

# Parse the script block + audit tag balance and ARIA reference integrity
node -e "
const html=require('fs').readFileSync('index.html','utf8');
new Function(html.split('<script>')[1].split('</script>')[0]);
const ids=new Set([...html.matchAll(/id=\"([^\"]+)\"/g)].map(m=>m[1]));
[...html.matchAll(/aria-controls=\"([^\"]+)\"|<label[^>]*for=\"([^\"]+)\"/g)]
  .forEach(m=>{const t=m[1]||m[2]; if(!ids.has(t))console.log('DANGLING:',t);});
console.log('ok');"
```

### Testing pure logic without a DOM

The JS lives in an IIFE with DOM access, so it cannot be required. To unit-test the pure parts
(`validators`, `scanCredentials`, `makeReference`, `SLA`), slice them out of the source by their
marker comments and `eval` them — this tests the real shipped source rather than a copy:

```bash
node -e "
const src=require('fs').readFileSync('index.html','utf8').split('<script>')[1];
const grab=(s,e)=>{const a=src.indexOf(s);return src.slice(a,src.indexOf(e,a));};
eval(grab('var CRED_WORD','function scanCredentials'));
eval(grab('function scanCredentials','var credState'));
console.log(scanCredentials('my password is hunter2').flagged);   // true
console.log(scanCredentials('the password reset link').flagged);  // false
"
```

## Architecture

Navigate by the marker comments — `grep -n "SECTION\|DESIGN TOKENS\|RESPONSIVE" index.html`.
CSS and JS blocks carry the same SECTION 1/2/3 numbering as the markup.

### Validator registry (the core pattern)

`validators` is an object keyed by **field element id**, each entry a function taking the field's
value and returning an error string or `''`. It is the single source of truth driving four
separate flows, all of which iterate `Object.keys(validators)`: blur/change validation wiring,
the submit loop, error clearing, and the "submit another ticket" reset.

**To add a field**, three things must line up or it will throw or silently skip:
1. an entry in `validators` keyed by the input's `id`;
2. an empty `<p class="error" id="<id>-error"></p>` in the markup;
3. `aria-describedby="<id>-error"` on the control.

`valueOf(id)` special-cases the three fields that are not a plain input: the `priority` radio
group, the `confirm` checkbox, and `attachment` (whose "value" is the result of
`checkAttachment()` — an extension/size check, since the file itself is never read). `controlOf(id)`
special-cases `priority`. Any new non-standard control needs a branch in both.

Error `<p>` elements are pre-existing and only have their `textContent` set or emptied — never
created or removed — so `aria-describedby` stays stable and `.error:empty{display:none}` handles
visibility. `showError()` also toggles `aria-invalid`.

### Credential guard

Two-tier by design: `scanCredentials()` returns `{flagged, blocking}`. A card-shaped number
(`CARD_PATTERN`) is unambiguous and **blocks** submit; everything else only shows the amber
`role="alert"` warning and still lets the ticket through.

`CRED_PATTERNS` requires a credential word, then a disclosure marker, then a lookahead asserting
the following token contains a digit, symbol or quote. **This tuning is deliberate and easy to
regress**: a looser pattern flags "I clicked the password reset link" and "my password is not
working" — the two most common real service-desk descriptions. If you touch these regexes,
re-run the corpus above with both the should-flag and should-be-clean cases.

### Other behaviour

- **Accordion**: custom buttons with `aria-expanded` + `hidden` panels, not `<details>`, because
  one-open-at-a-time, the chevron transition and roving Arrow/Home/End navigation all need
  explicit control. FAQ search caches each item's lowercased `textContent` in `dataset.search`
  on init, so **changing FAQ text at runtime will not update the search index**. Filtering closes
  any item it hides, and `visibleButtons()` skips filtered-out items in the arrow-key ring.
- **`[hidden]{display:none !important}`** sits at the top of the reset. Several panels that JS
  toggles via the `hidden` property carry a class with `display:flex` (`.warn`, `.note`), which
  out-specifies the UA `[hidden]` rule. Remove it and the credential warning is permanently
  visible. Any new JS-toggled panel relies on it.
- **Ticket reference**: `UOB-ITSD-YYYYMMDD-####` from local date + 4 random digits.
- **SLA map**: `{Critical:'1 hour', High:'4 hours', Medium:'1 business day', Low:'3 business days'}` —
  mirrored in prose in FAQ item 2; update both together.
- **Success panel**: swaps with the form via the `hidden` property and moves focus to the
  heading (`tabindex="-1"`).

### Security posture

Beyond the CSP meta tag: the page shows a red banner instead of the form if it is ever framed
(`frame-ancestors` is ignored in a meta CSP, so this is the substitute), `SLA` and `ALLOWED_EXT`
are frozen, and every user-supplied string — filenames included — reaches the DOM through
`textContent`. There is no `innerHTML`, `eval` or `document.write` in the file; a grep for them
should only ever hit prose and comments. The four claims in the Security section of the page are
each implemented, so do not add a fifth card without adding the control.

### Styling

All colours are `:root` custom properties — change tokens, not call sites. `--green` (#047857) is
the only green that clears AA on white; `--green-bright` and `--cyan` are for dark surfaces
(hero, footer, terminal) and for fills, never small text on white. Focus rings are never removed,
only restyled via `:focus-visible`. Breakpoints at `max-width:960px` and `max-width:768px`, plus a
`prefers-reduced-motion` block that zeroes every animation and forces `.reveal` visible.

### Animation

Decorative only, and all of it degrades to nothing under reduced motion:

- `.hero__grid` (drifting 48px grid) and `.hero__scan` (sweeping band) are absolutely-positioned
  `aria-hidden` spans behind the hero content.
- The hero terminal reveals its lines with staggered `animation-delay` on `.term__body > span`.
  **The `>` matters** — a descendant selector breaks the inline `.term__ok` / `.term__prompt`
  spans onto their own lines.
- `.reveal` elements start at `opacity:0` and are shown by an IntersectionObserver that also
  stamps a per-sibling `transitionDelay` for a wave stagger. If the observer is unavailable, or
  motion is reduced, the script adds `.is-in` to everything up front — the page is never left
  invisible because JS didn't run the way it expected.
