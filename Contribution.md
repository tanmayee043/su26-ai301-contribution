# Contribution [#]: Feature: Feature: Indicate errors to the user, if possible for saved replies in GitHub

**Contribution Number:** 1

**Student:** Tanmayee Maram

**Issue:** [\[GitHub issue link\]  ](https://github.com/JoshuaKGoldberg/refined-saved-replies)

**Status:** Phase II  Complete 
            (Forked the project, link - https://github.com/tanmayee043/AI301-Open-Source-First-Issue)
            
**Note**: I did not post an interest comment on the issue because this project's CONTRIBUTING.md explicitly prohibits issue-claiming comments - skipping it was intentional, to follow the maintainer's rules.

---

## Why I Chose This Issue
I chose this issue primarily because it is labeled as a "good first issue," which made it feel approachable as someone who is just getting started with open source contributions. On top of that, it was marked "status: accepting prs" with no one already assigned, so I felt confident I could dive in without stepping on anyone's work or waiting around for permission.

On the technical side, the fact that it is written in TypeScript was a big draw, since it builds on the JavaScript I am already comfortable with. It also involves working on a browser extension - modifying GitHub's own pages through a content script - which is something I had never done before and really wanted hands-on experience with. This felt like a good opportunity to learn how extensions inject into and interact with a live website in a real-world context, while fixing a genuine user-experience problem (the extension currently failing silently when it can't load a repository's replies).

---

## Understanding the Issue

### Problem Description

The refined-saved-replies extension adds a repository's shared replies (defined in a .github/replies.yml file) into GitHub's Saved Replies dropdown. The problem is that when something goes wrong - the replies file fails to load, or it exists but is malformed - the extension fails silently. It only writes an error to the browser's developer console, which almost no everyday user ever opens. So from the user's perspective, the repository's replies just don't show up, with no explanation of why. This issue asks for those failures to be made visible instead: show a small indication in the Saved Replies dropdown when the replies file can't be loaded, and log a console.error when the expected dropdown can't be found on the page.

### Expected Behavior

When a repository's `.github/replies.yml` can't be loaded or is invalid, the extension should show a small indication in the Saved Replies dropdown so the user knows something went wrong. If the saved-replies dropdown can't be found where it's expected, a `console.error` should be logged.

### Current Behavior

The extension fails silently. On any load/parse failure it only writes a `console.error` (which most users never see) and stops — so the repository's replies just don't appear, with no explanation.

### Affected Components

- `src/fetchRepliesConfiguration.ts` — fetches/validates the replies file; returns nothing on failure.
- `src/content-script.ts` — injects replies into the dropdown; silently returns when the config is missing.

---

## Reproduction Process

### Environment Setup

The project uses **pnpm** with a set of npm scripts documented in `.github/DEVELOPMENT.md`.
Setup on Windows (PowerShell) was straightforward:

- `pnpm install` - install dependencies
- `pnpm dev` - build the extension (esbuild bundles `src/content-script.ts` into `lib/content-script.js`)
- `pnpm test` (Vitest), `pnpm tsc` (type-check), `pnpm eslint` (lint)

The challenges I hit were minor but worth noting:
1. I initially didn't realize the TypeScript source has to be **built** into `lib/content-script.js`
   before the browser can run it — `manifest.json` points to the *built* file, not the `.ts` source.
   Running `pnpm dev` produces that file.
2. On PowerShell, `esbuild` prints its build summary to stderr, which PowerShell surfaces as a
   scary-looking `NativeCommandError` even though the build actually succeeds. I confirmed it worked
   by checking that `lib/content-script.js` was created and the output said "Done".

### Steps to Reproduce

Because this issue is about **silent error handling** rather than a crash, I reproduced the current
(pre-fix) behavior at the code/behavior level:

1. Clone the fork and run `pnpm install`.
2. Open `src/fetchRepliesConfiguration.ts`. On a failed or invalid fetch of `.github/replies.yml`,
   the function only calls `console.error(...)` and then returns `undefined`.
3. Open `src/content-script.ts`. When `fetchRepliesConfiguration` returns `undefined`, `main()`
   simply `return`s — nothing is rendered for the user.
4. **Expected:** the user should see some indication in the Saved Replies dropdown that the
   repository's replies couldn't be loaded.
5. **Actual:** nothing is shown. The only trace is a `console.error` in DevTools, which most
   users never open.

I also confirmed this via the pre-existing unit tests, which asserted the function returns
`undefined` on error / 404 / invalid input alike — i.e., no distinction between failure types and
no user-facing signal.

**Live-browser reproduction (limitation):** I built the extension and loaded it into Chrome
(`chrome://extensions` → *Load unpacked*), then opened GitHub issue/PR pages to try an end-to-end
reproduction. I found the extension **no longer runs on current GitHub at all due to change in GitHub UI**: its very first
selector —
`document.querySelector('[data-show-dialog-id="saved_replies_menu_new_comment_field-dialog"]')`
— returns `null`, because GitHub has redesigned its Saved Replies UI. So `content-script.ts` exits
at its first check before reaching any error-handling code, which means a full live demo of the
silent failure isn't currently possible. This is a pre-existing issue affecting the whole extension,
not something my change caused.

### Reproduction Evidence

- **Working branch:** https://github.com/tanmayee043/AI301-Open-Source-First-Issue/tree/fix/indicate-errors-to-user
- **My findings:** The silent-failure path lives entirely in `fetchRepliesConfiguration.ts`
  (`console.error` + `return`) and `content-script.ts` (early `return` on a missing value).
  Confirmed via code inspection and the existing unit tests. I also discovered the GitHub UI drift
  above, which prevents live reproduction.

---

## Solution Approach

### Analysis

The root cause is that **every failure mode collapses into the same empty result.**
`fetchRepliesConfiguration` returns `undefined` for a 404 (no file — normal), a non-404 network
error, *and* an invalid/malformed file. `content-script.ts` then treats any non-value as "silently
stop." Because the caller can't tell a normal missing file apart from a real failure, there's no way
to decide *when* to show something — so it shows nothing in every case.

### Proposed Solution

Make `fetchRepliesConfiguration` return a **typed, discriminated result** — `success` / `notFound` /
`error` — so the caller can distinguish a normal missing file (404, stay quiet) from a real failure
(show something). Then render a small indication in the Saved Replies dropdown only for real
failures, and add a `console.error` when the dropdown itself can't be found.

### Implementation Plan (UMPIRE)

**Understand:** When a repository's `.github/replies.yml` can't be loaded or parsed, the extension
fails silently (console-only), so users get no signal. It should instead show a small indication in
the dropdown, and a missing file (404) should stay quiet since most repos don't use the feature.

**Match:** Existing patterns I can build on — the current `console.error` calls in
`fetchRepliesConfiguration.ts`; the `createElement` + `"select-menu-divider"` pattern already used
in `content-script.ts` to build the "Repository replies" section; and the `isBodyWithReplies`
validation helper.

**Plan:**
1. In `fetchRepliesConfiguration.ts`, introduce a `RepliesConfigurationResult` discriminated union
   (`{ type: "success"; configuration }` | `{ type: "error"; message }` | `{ type: "notFound" }`)
   and return it: 404 → `notFound` (quiet), non-404 / invalid → `error` (keeping the `console.error`),
   valid → `success`.
2. Update `fetchRepliesConfiguration.test.ts` to assert the three new result shapes.
3. In `content-script.ts`, return early only on `notFound`; on `error`, render a small muted
   indication in the dropdown; on `success`, keep existing behavior. Add a `console.error` when the
   reply menus can't be found.

**Implement:** (Phase III) — commits will be pushed to
https://github.com/tanmayee043/AI301-Open-Source-First-Issue/tree/fix/indicate-errors-to-user

**Review:** Self-review against the project's `.github/CONTRIBUTING.md`: conventional-commits PR
title, fully fill out the PR template (including the 📨 emoji), keep the PR single-purpose, do **not**
post an issue-claiming comment (the guidelines prohibit it), and ensure status checks pass before
review.

**Evaluate:** Vitest unit tests covering the three result types (`success` / `notFound` / `error`),
plus `pnpm tsc` and `pnpm eslint` clean, and the full suite to catch regressions. Note: end-to-end
browser verification is blocked by the GitHub UI drift described above, so verification relies on the
automated tests.


---

## Testing Strategy

### Unit Tests

Updated `src/fetchRepliesConfiguration.test.ts` (Vitest) to cover all three result shapes:

- [x] Returns an `error` result **and** logs `console.error` when the fetch is not ok and status is not 404
- [x] Returns a `notFound` result **and** does *not* log when status is 404
- [x] Returns an `error` result **and** logs `console.error` when the fetched config is invalid/malformed
- [x] Returns a `success` result (with the parsed configuration) for a valid file

Full suite: **28 tests passing.** I also ran `pnpm tsc` (type-check) and `pnpm eslint` — both clean.

### Integration Tests

- The project has **no test file for `content-script.ts`** — it's DOM/browser code that the project
  itself does not unit-test — so I followed that existing convention and did not add one. The
  testable logic (the result types) lives in `fetchRepliesConfiguration`, which is fully covered.

### Manual Testing

- I built the extension (`pnpm dev`) and loaded it into Chrome (`chrome://extensions` → *Load
  unpacked*), then opened GitHub issue/PR pages to verify end-to-end.
- **Limitation:** the extension no longer runs on current GitHub — its first selector
  (`[data-show-dialog-id="saved_replies_menu_new_comment_field-dialog"]`) returns `null` because
  GitHub redesigned its Saved Replies UI, so `content-script.ts` exits before reaching my code. I
  confirmed this in DevTools (the selector query returned `null`). Because a live demo isn't
  possible, verification relies on the automated tests above, and I flagged the UI drift to the
  maintainer in my PR.

---

## Implementation Notes

### Week 3 Progress

**What I built:**
- Reworked `src/fetchRepliesConfiguration.ts` so it returns a **typed, discriminated result**
  instead of "the configuration or `undefined`". The new `RepliesConfigurationResult` is one of:
  - `{ type: "success"; configuration }`
  - `{ type: "error"; message }`  (non-404 network failure, or invalid/malformed YAML)
  - `{ type: "notFound" }`         (404 — repo simply has no replies file; stay quiet)
- Updated `src/content-script.ts` to act on that result:
  - `notFound` → return early and show nothing (normal for most repos)
  - `error` → render a small muted indication in the Saved Replies dropdown
    ("Couldn't load this repository's replies: …")
  - `success` → unchanged existing behavior
  - Added a `console.error` when the saved-replies dropdown menus can't be found on a page
    where they're expected (the issue's second request).
- Kept the existing `console.error` logging in place — it's still useful for developers, and the
  issue asked to *add* user-facing signaling, not remove logging.

**Challenges faced:**
- **Keeping each commit green.** Changing the return type of `fetchRepliesConfiguration` immediately
  broke `content-script.ts` at compile time (`tsc` error: `Property 'replies' does not exist on
  type RepliesConfigurationResult`). Rather than one big commit, I split the work so every commit
  compiles and passes tests (see Code Changes below) — a behavior-preserving refactor first, then
  the feature.
- **ESLint union ordering.** The project uses a strict `perfectionist/sort-union-types` rule; my
  first ordering of the union members failed lint. The linter told me the exact expected order, so
  I reordered `success | error | notFound`.
- **Pre-commit hook.** The repo runs `lint-staged` on commit, which auto-runs Prettier on staged
  files. Good to know so formatting stays consistent — my commits were reformatted automatically.
- **TypeScript narrowing.** In `content-script.ts` I used an `if (result.type === "error") { …;
  continue; }` inside the render loop, which lets TypeScript narrow the rest of the loop to
  `success` so `result.configuration.replies` type-checks cleanly.

### Code Changes

- **Files modified:**
  - `src/fetchRepliesConfiguration.ts` — new `RepliesConfigurationResult` union + return logic
  - `src/fetchRepliesConfiguration.test.ts` — updated tests for the three result shapes
  - `src/content-script.ts` — consume the result, render the error indication, add the `console.error`
- **Key commits:**
  - `d516875` — `refactor: return a typed result from fetchRepliesConfiguration`
    (behavior-preserving; also updates the tests so the commit stays green)
    https://github.com/tanmayee043/AI301-Open-Source-First-Issue/commit/d516875
  - `7ae184f` — `feat: indicate replies loading errors to the user`
    (the actual user-facing behavior + the missing-dropdown `console.error`)
    https://github.com/tanmayee043/AI301-Open-Source-First-Issue/commit/7ae184f
- **Approach decisions:**
  - **Discriminated union** so the caller can tell a normal missing file (404 → quiet) apart from a
    real failure (→ show something). This was the key enabler for the whole feature.
  - **Two commits (refactor → feature)** so each commit independently compiles and passes tests,
    which makes the history easy to review.
  - **Reused existing UI patterns** (`createElement`, the `select-menu-divider` header) and Primer
    utility classes (`color-fg-muted`, `px-3 py-2`) for the indication, to match the extension's
    existing style rather than inventing new markup.

---

## Pull Request

**PR Link:** https://github.com/JoshuaKGoldberg/refined-saved-replies/pull/1444

**PR Description:**

*What does this PR do?:* Surfaces errors loading a repository's `.github/replies.yml` to the user
instead of failing silently. `fetchRepliesConfiguration` now returns a typed result
(`success` / `notFound` / `error`); on a real load or parse failure, a small indication is shown in
the Saved Replies dropdown, and a `console.error` is logged when the dropdown can't be found on a
page where it's expected. A missing file (404) is still treated as normal and stays quiet.

*Why was this PR needed?:* Issue #2 asked for errors to be indicated to the user. Previously the
extension only logged failures to the developer console and returned, so an everyday user got no
signal when a repository's replies couldn't be loaded — the replies just silently didn't appear.

*What are the relevant issue numbers?:* Closes #2

*Does this PR meet the acceptance criteria?:*
- [x] Addresses an existing open issue (fixes #2)
- [x] Issue was marked `status: accepting prs`
- [x] Steps in CONTRIBUTING.md were followed (conventional-commits title, PR template, 📨)
- [x] Unit tests updated for the new result type; all tests passing (28)
- [x] `pnpm tsc` and `pnpm eslint` clean
- [ ] Live browser verification — **blocked by GitHub's Saved Replies UI redesign** (documented in
      the PR and my README); verification relies on the automated tests

**Maintainer Feedback:**
- **Jun 24:** On opening the PR, the automated **OctoGuide** best-practices bot flagged that one
  PR-template checkbox ("That issue was marked as `status: accepting prs`") wasn't checked in the
  raw markdown. Root cause: I had simplified that template line and dropped its link, so the bot
  couldn't match it to the template task. I fixed it by restoring the exact template text (with the
  link) and keeping the box checked.
- **Jun 25:** OctoGuide re-ran automatically after my edit and posted *"All reports are resolved
  now."*
- **Ongoing:** The 11 required CI checks are **awaiting maintainer approval** — a standard gate for
  first-time contributors — so they haven't run yet, and there's been no human review so far.
- **Note:** Per the project's CONTRIBUTING.md, I intentionally did **not** @-mention or tag a
  maintainer to request review; the guidelines explicitly ask contributors not to.

**Status:** Awaiting review

---

## Learnings & Reflections

### Technical Skills Gained
- Modeling outcomes with **TypeScript discriminated unions**, and how that lets a caller handle
  distinct cases (success / not-found / error) cleanly and type-safely.
- How a **browser extension** actually works: content scripts, the `manifest.json` matches, and the
  build step that bundles TypeScript into the `lib/content-script.js` the browser runs.
- **Vitest** unit testing (and why the project's `console-fail-test` setup means console calls must
  be mocked), plus keeping `tsc` and ESLint green.
- The real **fork → branch → commit → PR** open-source workflow, conventional-commit messages, and
  reading a project's CONTRIBUTING.md *before* coding.

### Challenges Overcome
- **Keeping every commit green:** changing a return type broke another file at compile time, so I
  learned to sequence the work as a behavior-preserving refactor first, then the feature.
- **Diagnosing the GitHub UI drift:** when the extension showed no effect, I used DevTools to test
  the selector directly (`querySelector(...) === null`) and traced it to GitHub redesigning its
  Saved Replies UI — a good lesson in how DOM-scraping extensions break when the host site changes.
- **The OctoGuide bot:** learning that automated reviewers match PR text literally against the
  template, and that small deviations (a dropped link) can fail a check.

### What I'd Do Differently Next Time
- **Verify the target actually runs first:** I'd confirm the extension still works on current GitHub
  *before* planning to rely on live testing — that would have surfaced the UI drift on day one.
- **Match the PR template exactly from the start** (links and all) to avoid the bot flag.
- Keep the PR description's acceptance-criteria checklist in mind while coding, so I'm tracking the
  right evidence as I go rather than reconstructing it at submission time.

---

## Resources Used

- [Link to helpful documentation]
- [Tutorial or Stack Overflow post that helped]
- [GitHub issues or discussions that helped]
