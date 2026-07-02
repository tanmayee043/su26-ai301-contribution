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

[What should happen?]

### Current Behavior

[What actually happens?]

### Affected Components

[Which parts of the codebase are involved?]

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

- [ ] Test case 1: [Description]
- [ ] Test case 2: [Description]
- [ ] Test case 3: [Description]

### Integration Tests

- [ ] Integration scenario 1
- [ ] Integration scenario 2

### Manual Testing

[What you tested manually and results]

---

## Implementation Notes

### Week [X] Progress

[What you built this week, challenges faced, decisions made]

### Week [Y] Progress

[Continue documenting as you work]

### Code Changes

- **Files modified:** [List]
- **Key commits:** [Links to important commits]
- **Approach decisions:** [Why you chose certain approaches]

---

## Pull Request

**PR Link:** [GitHub PR URL when submitted]

**PR Description:** [Draft or final PR description - much of the content above can be adapted]

**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** [Awaiting review / Iterating / Approved / Merged]

---

## Learnings & Reflections

### Technical Skills Gained

[What you learned technically]

### Challenges Overcome

[What was hard and how you solved it]

### What I'd Do Differently Next Time

[Reflection on your process]

---

## Resources Used

- [Link to helpful documentation]
- [Tutorial or Stack Overflow post that helped]
- [GitHub issues or discussions that helped]
