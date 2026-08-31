# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-page training/demo mockup of an internal IT Support Service Desk, branded "UOB Bank"
for classroom use. The entire project is one file: `index.html` (~1,070 lines) containing the
markup, a `<style>` block, and a `<script>` block.

## The hard constraints

These are the point of the exercise, not incidental. Breaking any of them breaks the deliverable:

- **One self-contained file.** No frameworks, no build step, no package manager, no external
  requests. It must work by double-clicking from `file://`.
- **No persistence, no network.** `localStorage`, `sessionStorage`, `fetch`, and
  `XMLHttpRequest` are forbidden. Submitted tickets are pushed onto the in-memory `tickets`
  array and are intentionally lost on reload.
- **No external assets.** The only SVGs are inline; the select chevron is a `data:` URI. The
  sole allowed `http` string in the file is the SVG namespace `www.w3.org/2000/svg`.
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

`valueOf(id)` and `controlOf(id)` special-case the two fields that are not a plain input: the
`priority` radio group and the `confirm` checkbox. Any new non-standard control needs a branch
in both.

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
  one-open-at-a-time and the chevron transition need explicit control. FAQ search caches each
  item's lowercased `textContent` in `dataset.search` on init, so **changing FAQ text at runtime
  will not update the search index**. Filtering closes any item it hides.
- **Ticket reference**: `UOB-ITSD-YYYYMMDD-####` from local date + 4 random digits.
- **SLA map**: `{Critical:'1 hour', High:'4 hours', Medium:'1 business day', Low:'3 business days'}` —
  mirrored in prose in FAQ item 2; update both together.
- **Success panel**: swaps with the form via the `hidden` property and moves focus to the
  heading (`tabindex="-1"`).

### Styling

All colours are `:root` custom properties — change tokens, not call sites. Gold (`--gold`) is an
accent for borders/fills only and never small text on white; it fails AA at body size. Focus
rings are never removed, only restyled via `:focus-visible`. One breakpoint at `max-width:768px`
collapses every grid to a single column, plus a `prefers-reduced-motion` block.
