---
description: "Generate publication-ready CHANGES file entries from git log using nginx changelog conventions. Never fabricate; flag uncertainty with [REVIEW NEEDED]."
tools: [read, search, execute, vscode_askQuestions]
---

# Changelog Generation

Produce concise, accurate plain-text changelog entries from the selected git range. Optimize for turnaround: minimize tool calls and round-trips — every extra command or question is time the user is waiting. Use the repository's changelog conventions for classification and brevity, but do not generate XML or require a translation. Every fact comes only from the current git log window.

## Workflow

1. `git tag --sort=-v:refname | head -20`. In one `vscode_askQuestions` call, ask two questions at once — never split across two round-trips: (a) the previous release tag, tags as clickable options, most recent recommended; (b) the current boundary, options `["HEAD (latest unreleased)", "<the most recent tag>"]` with `allowFreeformInput: true` so the user can type any other tag, "HEAD" recommended. Each question must have at least two options (or `allowFreeformInput: true`) to avoid a tool error.
2. `git log <prev>..<current> --stat` once. This output already contains the full commit body (subject + message) for every commit — treat it as the source of truth for symptom wording, fault type, and exact log/alert strings. Do not re-run `git show`/`git log -p` per commit to re-fetch bodies already visible in this output; only fall back to a targeted `git show <hash>` when the `--stat` body is genuinely insufficient to classify or word a specific bullet (e.g. missing symptom detail).
3. Version: `NGINX_VERSION` from `src/core/nginx.h` at current; flag if no "Version bump" commit.
4. Date: tag date if current is a tag; else `[REVIEW NEEDED]`.
5. Classify → group → output in one pass. Return only the plain-text release block, without a preamble or summary.

## Skip

Docs, tests, CI, refactors, merges, version bumps, build/release infrastructure, and comment-only changes. A fix to nginx that restores building on a named supported OS or compiler, or with an already-supported dependency, is user-visible and is not build infrastructure; include it as a Bugfix. Groundwork whose only consumer is a feature introduced in this same range (the feature bullet already covers it). Fixes justified only by an internal security/fuzzing tracker with no public CVE, or co-authored by an automated patching bot — these go through separate security disclosure, not CHANGES. Changes with no user-facing surface at all (no directive/module/variable/protocol behavior an admin could observe or configure), even when framed as a "Change".

Regression note ("the bug had appeared in X.Y.Z"): add only when the commit itself flags a regression (says "regression"/"broke", or cites another commit/version). A cited bare hash needs resolving to a version: `git tag --contains <hash> --sort=v:refname | head -1`. If that hash is old (several releases back), it isn't a recent regression — it's a long-standing gap; drop the note and classify as a Feature instead (see Classify).

## Classify

- **Security**: CVE or clear exploit. Include CVE ID and reporter.
- **Feature**: new directive/module/variable; new compatibility with a dependency version that didn't previously exist; or an old, always-missing capability finally closed. Minimal announcement.
- **Change**: behavior/default/deprecation shift with real user impact. State the bare fact only, present tense, past participle ("has been removed", "now defaults to") — never append a rationale/justification clause (no "as it was never finished", "becauseX", "since Y"); compare changes.xml: "HTTP/2 server push support has been removed.", "obsolete aio and rtsig event methods have been removed." — nothing more.
- **Bugfix**: a user-visible behavior that was broken, including nginx failing to build on a named OS or compiler, or with an already-supported dependency. State the affected behavior or component; for example, "nginx could not be built on <OS>." A platform-specific fix is not automatically a Workaround.
- **Workaround**: nginx-side mitigation for a defect in an external library, OS, or compiler, as identified in the commit body. Name the affected external component when stated; do not use this category just because a fix is platform-specific.

Order: Security → Features → Changes → Workarounds → individual Bugfixes → optional grouped `Bugfixes in ...` line last.

## Wording

One short sentence per bullet. Roman and Sergey keep CHANGES terse — when in doubt, cut, don't elaborate.

- Prefix each entry with its classification (`Feature`, `Change`, `Bugfix`, `Workaround`, or `Security`). Name a module or directive tersely for features, without a directive list or syntax/config snippets. Existing directives gaining a capability may state it briefly.
- Bugfix wording is normally terse, such as `in the ngx_http_dav_module.` or `in memory allocation error handling.` Do not invent a symptom or trigger.
- Build failures and other concrete user-visible failures should name the failed operation and stated platform or trigger; do not hide them in a generic module summary.
- Write a full symptom + trigger sentence only when the bug is a crash/hang/security issue/data corruption reachable under an ordinary, common condition, or when the commit cites a regression ("the bug had appeared in X.Y.Z"). Exactly one symptom clause + one trigger clause, nothing else. Use the fault type stated in the body verbatim (e.g. "a use-after-free"); don't tack on a downstream consequence, and don't guess an unstated fault — unless the pattern described has a universally known consequence (e.g. an unvalidated descriptor into `FD_SET()` → segfault), in which case state that consequence in its conventional phrasing.
- Credit (`Thanks to <name>.`) can appear on either a terse or a full bugfix — use the body's "Reported by" name if present, else the author if their email is outside the core team (@nginx.com/@f5.com).
- For a missing field-copy/init bug, check what the consuming code does with the missing value instead of guessing ("were not available" vs. "returned incorrect values").
- Quote alerts/log lines verbatim when the body includes one — still counts as the one symptom clause.
- "might" for races/timing. "or" only to join two *different commits'* symptoms in a same-root-cause group — never two effects from one commit.
- Prefer public names (HTTP/2, FastCGI, control API) over module/function names; quote directive-like values the same as directives (e.g. `"select"`). Variables get `$`, never quotes.
- Never name internal C functions/APIs (`ngx_memcpy()`, `i2d_SSL_SESSION()`, `FIOASYNC`, etc.) in the prose, even if the commit body does — describe the user-visible effect instead. The one exception is a verbatim quoted log/alert string that itself happens to contain a function name (e.g. `"ngx_slab_alloc() failed: no memory"`).
- Omit anything not stated in the body — never infer specifics.

## Grouping

Default: every qualifying commit gets its own bullet — don't fold into a combined line or replace specific fixes with a summary paragraph. Match the short, concrete entries in changes.xml.

- Same root cause across multiple commits → merge into one bullet, leading with the most concrete stated symptom (full form, not terse).
- Sibling fixes (parallel modules, same bug shape) → one bullet naming both, no per-module detail.
- Reserve `Bugfixes in <Module>, <Module>, and <Module>.` (public component names, no symptom text) for leftover, genuinely minor fixes only — use at most once per release, only after writing individual entries for substantive fixes, and never as a substitute for a crash/hang/security/regression or concrete build-failure entry. Do not combine unrelated components into this line merely to shorten the output.
- Credit `Thanks to <name>.` applies to any bugfix, terse or full, when the body names a reporter, or the patch author's email is outside the core team (@nginx.com/@f5.com) — credit whoever the body identifies as the true patch author, not a requester or sign-off-only committer.

## Plain-Text Format

```
Changes with nginx <ver>                                     <D Mon YYYY>

	*) Feature: the "<name>" directive.

	*) Bugfix: in the <module>.

	*) Bugfix: <symptom>; <trigger>.
```

- Header includes the version and date. Use the current tag date when the boundary is a tag; for an untagged boundary, write `[REVIEW NEEDED]` in the date position rather than inventing a release date.
- Leave a blank line between entries. Indent each entry four spaces; wrap long entries with a seven-space continuation indent.
- Preserve public module names, quote directives and headers, and prefix variables with `$`.
- Do not use XML tags, language pairs, or Markdown fences.
- Default to the terse bugfix form; reserve a full symptom-and-trigger sentence for crash/hang/security/corruption or regression-cited bugs.
