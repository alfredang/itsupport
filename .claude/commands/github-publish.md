---
description: Security-scan, push to GitHub, capture screenshots, write the README, deploy GitHub Pages via Actions, and fill in the About section
argument-hint: [github repo URL or owner/name]
allowed-tools: Bash, Read, Write, Edit, Glob, Grep, mcp__playwright__browser_navigate, mcp__playwright__browser_resize, mcp__playwright__browser_take_screenshot, mcp__playwright__browser_close
---

# Publish this project to GitHub

Target repository: **$ARGUMENTS**

If no repository was supplied above, check for an existing `origin` remote and use that. If
there is neither, stop and ask the user for the repo before doing anything else.

Take the project from its current state to fully published: security-scanned, pushed, README
accurate and illustrated, Pages deploying via Actions, About section filled in, and the live
URL verified and linked back.

## Step 1 — Security scan (blocking gate, runs BEFORE the first push)

Never reorder this after the push. Once a secret reaches a public remote it must be **rotated
at the provider**, not merely deleted — the object survives in the remote's history and in
every fork, clone and cache.

- `git status --porcelain` then `git diff --staged` — review the real diff, not filenames.
- Ensure `.gitignore` covers `.env`, `.env.*` (with `!.env.example`), `*.pem`, `*.key`,
  `*.p12`, `*.pfx`, `*.keystore`, `id_rsa*`, and local credential stores. Add before committing.
- Confirm nothing sensitive is already tracked:
  `git ls-files | grep -iE '\.(env|pem|key|p12|pfx|keystore)$|secret|credential'`
- Grep the whole tree for credential patterns — API keys, bearer tokens, `ghp_` /
  `github_pat_` GitHub tokens, AWS `AKIA` IDs, private-key PEM headers, Slack `xox*` tokens,
  `sk-` provider keys, and connection strings with an inline password.
- If the repo already has history, search that too — a secret gone from HEAD can still be
  reachable in an earlier commit about to be pushed: `git log -p -S PATTERN --all`
- List every real email address and phone number and confirm each is intended to be public:
  `grep -rInoE "[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}" . | sort -u`
- Check non-obvious leak channels: a token in a commit message, a credential hardcoded in a
  workflow instead of `secrets.*`, a real key committed as an `.env.example` value.

**If anything is found: stop, do not push.** Report the file, whether it is only in the working
tree (gitignore it) or already committed (history rewrite needed), and whether it was ever
pushed (then it must be rotated at the provider). Ask how to proceed — never rewrite published
history unprompted.

Also confirm intended visibility with the user before a public push when the content includes
third-party branding, internal-looking data, or anything they may not want indexed.

## Step 2 — Push

**Try the push before concluding that auth is missing.** `gh auth status` reporting logged out
does NOT mean the push will fail — Git Credential Manager often holds a working credential
independently of `gh`. Attempt it first:

```bash
GIT_TERMINAL_PROMPT=0 git push -u origin main
```

- Run `git init`, `git branch -M main`, `git remote add origin <url>` first if needed.
- Stage deliberately and review; write a commit message explaining *why*, matching `git log` style.
- If the remote has commits you lack, pull and reconcile. Never force-push without explicit consent.
- Only if the push genuinely fails on credentials: `gh auth login` is **interactive and cannot
  be run by the agent** — ask the user to run it in their own terminal. Scopes `repo` +
  `workflow`; `workflow` is required whenever the commit touches `.github/workflows/`.

## Step 3 — Screenshots

A reader decides whether to click the live demo from the README image. Capture the real
rendered page with the Playwright MCP browser — never mock one up, never describe a screenshot
you did not take.

- **Skip only when there is nothing visual** — a library, a CLI with no TUI, a bare config
  repo. Say so in the report rather than silently omitting this step.
- **Point the browser at the local source**, so the shot matches the commit being pushed rather
  than whatever is currently deployed: `browser_navigate` to `file:///<abs path>/index.html`
  for a static site, or to the dev server for anything that needs one. On Windows the
  `file://` form takes forward slashes and the drive letter: `file:///c:/Users/...`.
- `browser_resize` to **1440×900** first. The default viewport is narrower and triggers the
  mobile breakpoint on responsive layouts, which is not the shot you want at the top of a README.
- Write into `docs/` — **create the directory first**, since `browser_take_screenshot` fails
  with `ENOENT` rather than creating it.
- Take two: `docs/hero.png` (viewport only) as the lead image, and `docs/screenshot.png`
  (`fullPage: true`) for the whole page. A full-page shot of a long page renders as an
  unreadable sliver at README width — put it behind a `<details>` block, not inline.
- **Read each PNG back before committing it.** Confirm it shows the finished page and not a
  spinner, an error overlay, an unstyled flash or a cookie banner. Re-shoot if it does.
- Drive real UI state where the interesting behaviour is not visible at rest — fill a form, open
  an accordion, trigger a validation error — rather than shipping only the empty default view.
- Give every image **descriptive alt text** saying what is in the frame. `![screenshot](...)`
  is useless to a screen reader and to anyone whose images fail to load.
- Ensure `.playwright-mcp/` (traces, downloads, stray shots) is gitignored; commit only the
  files under `docs/`.
- `browser_close` when finished.

Reference the images from the README in the next step, with paths **relative to the repo root**
(`docs/hero.png`) so they resolve both on github.com and in a local Markdown preview.

## Step 4 — README

Read the actual source first; describe what this code does, not what such projects usually do.
Update in place if one exists — preserve what is still accurate. Cover: what it is; the
screenshots from Step 3 near the top; any placeholder or fictional content a reader could
mistake for real; how to run it, with the real command for this platform; required
configuration and what breaks if only one copy of a value is changed; how deployment works;
conventions a contributor could violate by accident; and how to test, or an explicit note that
there is no test suite.

## Step 5 — Pages via GitHub Actions

Check `.github/workflows/` first; do not duplicate an existing deploy. Otherwise create one
triggering on push to the default branch plus `workflow_dispatch`, with least-privilege
`contents: read` / `pages: write` / `id-token: write`, a `pages` concurrency group with
`cancel-in-progress: false`, then `actions/configure-pages` → `actions/upload-pages-artifact`
(repo root for a no-build static site, else the build output directory) → `actions/deploy-pages`.

**Known bootstrap failure — expect it on a repo that has never had Pages enabled.**
`configure-pages` with `enablement: true` fails with `Must have admin rights to Repository`,
and the run dies at that step with upload and deploy skipped. Confirm via `has_pages` on
`GET /repos/{owner}/{repo}`. Fix by enabling Pages directly with an admin credential (Step 6),
then re-running the failed run:

```bash
curl -s -X POST -H "Authorization: Bearer $TOKEN" -H "Accept: application/vnd.github+json" \
  https://api.github.com/repos/OWNER/REPO/pages -d '{"build_type":"workflow"}'

curl -s -X POST -H "Authorization: Bearer $TOKEN" \
  https://api.github.com/repos/OWNER/REPO/actions/runs/RUN_ID/rerun
```

Then poll the run until it reports **success** — never assume. If it fails, fetch the failing
step from `GET /actions/runs/{id}/jobs` and report the actual error.

## Step 6 — About section

Prefer `gh repo edit` when `gh` is authenticated. When it is not, the credential that
authenticated the push can be reused for the API — it is the user's own credential for the
same host:

```bash
TOKEN=$(printf "protocol=https\nhost=github.com\n\n" | git credential fill | sed -n 's/^password=//p')
```

**Never echo, log, interpolate into visible output, or commit that value.** Confirm it has admin
rights first via `permissions.admin` on `GET /repos/{owner}/{repo}`.

- `PATCH /repos/{owner}/{repo}` — `description`, `homepage`
- `PUT /repos/{owner}/{repo}/topics` — `names` array, lowercase-hyphenated, max 20

Write a description saying concretely what the project is and what it is built with. Choose
topics someone would actually search: language, stack, project type, hosting, notable libraries.

## Step 7 — Live link

- Read the real URL from `GET /repos/{owner}/{repo}/pages` (`html_url`) — never guess the
  `owner.github.io/repo` pattern, since a CNAME changes it.
- Set it as `homepage`, add it near the top of the README, then commit and push.
- **Verify it serves**: fetch the URL, confirm HTTP 200, and grep the response for real page
  content so a 404 stub cannot pass as success. Compare byte size against the local file.
  First deploys take a minute or two — confirm the run finished before reporting a 404 as broken.
- Confirm the README images resolve on the deployed repo page too — a screenshot committed
  after the README references it, or one left inside an ignored directory, shows as a broken
  image on github.com.

## Platform notes (Windows / Git Bash)

- Git Bash `/tmp` and Node resolve differently — Node reads `/tmp/x` as `C:\tmp\x`. Write temp
  files to the scratchpad or the repo, or pipe between commands instead of going via `/tmp`.
- Non-ASCII in a `curl -d` payload can be mangled into a 400. Write the JSON to a file with a
  quoted heredoc and send it with `--data-binary @file`.
- Heredocs containing nested quotes often break when passed through the Bash tool; use the
  Write tool for large files with mixed quoting.
- `gh auth login`, `git rebase -i`, and anything else interactive cannot be run by the agent.

## Report back

State the security scan result, including anything deliberately kept out of the repo; what was
pushed and at which commit; which screenshots were captured and verified, or why the step was
skipped; the live URL and that it was verified serving real content; the description and topics
set; and any step skipped or failed, with the reason. Never report success for a step that was
not verified.
