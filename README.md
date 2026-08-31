# IT Support Service Desk — Training Demo

**Live demo: <https://alfredang.github.io/itsupport/>**

A single-page internal IT Service Desk portal: ticket submission with full client-side
validation, a searchable FAQ accordion, and a mock ticket-reference generator. Built as a
teaching example of accessible, dependency-free front-end work — a dark-green terminal
aesthetic, CSS-only motion, and a Content Security Policy that makes "nothing leaves the
browser" enforceable rather than merely claimed.

![The IT Service Desk landing page: a dark green hero with a drifting grid backdrop, the headline "IT Support, Engineered for Speed", Submit a Ticket and Browse FAQs buttons, and a mock terminal panel showing a ticket being routed; the WhatsApp chat widget is open bottom-right with five suggested questions; three white cards below list the hotline, operating hours and average response time](docs/hero.png)

<details>
<summary>Full page — ticket form, credential guard and FAQ accordion</summary>

![Full-length page screenshot: the hero and contact cards, a four-card band explaining the page's security controls, the ticket form part-filled and showing a staff-ID validation error plus the amber credential warning, the FAQ accordion with its third item expanded, the dark four-column footer, and the WhatsApp widget open over the hero](docs/screenshot.png)

</details>

> **Training demo — not affiliated with or endorsed by United Overseas Bank Limited.**
> Every name, hotline, email address, SLA and ticket reference on this page is fictional.
> Nothing is submitted anywhere: there is no backend, no analytics and no network request
> of any kind. Do not enter real personal data into it.

## Running it

There is no build step, no package manager and no dependencies. Open the file:

```bash
start index.html      # Windows
open index.html       # macOS
xdg-open index.html   # Linux
```

It is designed to work correctly straight from `file://` — if it ever needs a web server,
something has been added that shouldn't have been.

## What's in it

One file, `index.html` (~1,600 lines): markup, a `<style>` block and a `<script>` block,
divided by `SECTION` marker comments that use matching numbering across all three.

- **Ticket form** — nine fields with per-field validation on blur and on submit, inline
  errors, `aria-invalid`, and focus sent to the first invalid field. On success it
  generates a reference of the form `UOB-ITSD-YYYYMMDD-####` and shows the SLA for the
  chosen priority.
- **Credential guard** — the description field is scanned for anything resembling a
  password, OTP, PIN or card number, and warns the user rather than accepting it. The
  form deliberately collects no credentials of any kind.
- **FAQ accordion** — eight items, one open at a time, built from buttons with
  `aria-expanded` rather than `<details>` so the chevron transition, the single-open rule and
  Arrow/Home/End navigation between headers can all be controlled explicitly. Live keyword
  search filters over both questions and answers, with a no-results state.
- **Security controls** — a `default-src 'none'` Content Security Policy in `<head>`, a
  clickjacking banner if the page is ever framed, attachment type/size checks, and a rule
  that every user-supplied string (filenames included) reaches the DOM via `textContent`.
  The four claims in the page's own Security section are each actually implemented.
- **WhatsApp widget** — a floating launcher bottom-right opens a panel of suggested
  questions, each of which hands off to WhatsApp with the message pre-filled. The chips and
  the CTA are plain `<a href>` links, so the widget works with JavaScript disabled; the script
  only adds open/close, focus handling, and Escape / outside-click dismissal. It is labelled
  as the course trainer's contact, not a bank support channel, and that disclaimer is pinned
  outside the scroll area so it stays visible on short screens.
- **Motion** — a drifting grid and a scanning sweep behind the hero, a terminal panel that
  types itself in, a scroll-reveal stagger driven by `IntersectionObserver`, and hover lifts
  on cards and buttons. All of it is decorative and all of it is switched off by
  `prefers-reduced-motion`; if the observer is unavailable the content is shown immediately
  rather than left invisible.

## Design constraints

These are the exercise, not incidental details. Changes that break them defeat the point:

| Constraint | Why |
| --- | --- |
| One self-contained file | Must run by double-clicking, with no build step or server |
| No `localStorage` / `sessionStorage` | Submitted tickets live in an in-memory array and are intentionally lost on reload |
| No `fetch` / `XMLHttpRequest` | Nothing the user types may leave the browser. The one allowed external URL is the WhatsApp widget's `wa.me` link, which is a navigation the *user* chooses, not a request the page makes |
| No external assets | Icons are inline SVG; the select chevron is a `data:` URI. This rules out Google Fonts — the type stacks name real families first and fall back to platform fonts |
| The CSP meta tag stays | It is what makes "no network" enforceable; the page's Security section describes it, so the two must move together |
| No credential collection | And the description field actively warns against pasting one |
| Footer disclaimer stays visible | It is what makes the bank branding acceptable |

Verify the first four at any time:

```bash
grep -inE "localStorage|sessionStorage|fetch\(|XMLHttpRequest|https?://" index.html \
  | grep -vE "www[.]w3[.]org/2000/svg|wa[.]me/"
```

The only expected match is the source comment that names them. Two strings are allowlisted:
the inline-SVG namespace and the WhatsApp widget's `wa.me` links.

## Conventions worth knowing before editing

- **Adding a form field takes three matching pieces**: an entry in the `validators` object
  keyed by the input's `id`, an empty `<p class="error" id="<id>-error">` in the markup, and
  `aria-describedby="<id>-error"` on the control. The registry drives validation, submission,
  error clearing and reset — miss a piece and the field is silently skipped.
- **Colours come from `:root` custom properties.** Change the token, not the call site.
  `--green` (#047857) is the only green that clears AA on white; `--green-bright` and `--cyan`
  belong on dark surfaces and in fills, never as small text on white.
- **`[hidden]{display:none !important}` at the top of the reset is load-bearing.** Panels that
  JS toggles via the `hidden` property carry classes with `display:flex` (`.warn`, `.note`),
  which out-specify the UA `[hidden]` rule. Remove it and the credential warning shows
  permanently.
- **`.term__body > span` needs the child combinator.** A descendant selector breaks the inline
  `.term__ok` / `.term__prompt` spans in the hero terminal onto their own lines.
- **Focus rings are never removed**, only restyled through `:focus-visible`.
- **The credential regexes are deliberately tuned** to ignore "my password is not working"
  and "the password reset link" — the two most common real service-desk phrasings. Loosening
  them reintroduces constant false positives.
- **The SLA table appears twice** — in the `SLA` object and in prose in FAQ item 2. Update both.
- **The WhatsApp number appears seven times** — six `wa.me` hrefs plus the widget's visible
  disclaimer. `grep -c "6596983731|9698 3731" index.html` should return 7; change them together.

## Testing

There is no test framework. The script is wrapped in an IIFE and touches the DOM, so it
cannot be imported; to exercise the pure logic, slice it out of the source and evaluate it,
which tests the real shipped code rather than a copy:

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

The same approach works for `validators`, `makeReference` and the `SLA` map.

`CLAUDE.md` holds the fuller architecture notes.

## Deployment

Pushing to `main` triggers `.github/workflows/pages.yml`, which publishes the repository
root to GitHub Pages. There is nothing to build, so the workflow uploads the root directly.
