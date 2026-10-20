---
description: "Generate publication-ready CHANGES file entries from git log using nginx changelog conventions. Never fabricate; flag uncertainty with [REVIEW NEEDED]."
tools: [read, search, execute, vscode_askQuestions]
---

# Changelog Generation

Produce concise, accurate plain-text changelog entries from the selected git range. Optimize for turnaround: minimize tool calls and round-trips — every extra command or question is time the user is waiting. Use the repository's changelog conventions for classification and brevity, but do not generate XML or require a translation. Every fact comes only from the current git log window.

## Workflow

1. `git tag --sort=-v:refname | head -20`. In one `vscode_askQuestions` call, ask two questions at once — never split across two round-trips: (a) the previous release tag, tags as clickable options, most recent recommended; (b) the current boundary, options `["HEAD (latest unreleased)", "<the most recent tag>"]` with `allowFreeformInput: true` so the user can type any other tag, "HEAD" recommended. Each question must have at least two options (or `allowFreeformInput: true`) to avoid a tool error.
2. `git log <prev>..<current> --format='commit %h%nAuthor: %an <%ae>%n%s%n%b' --stat` once. This provides full commit bodies, author emails, and file stats — use it as the source of truth for classification, wording, credits, and exact log/alert strings. Do not re-run `git show`/`git log -p` per commit to re-fetch bodies already visible; only use a targeted `git show <hash>` when the body is genuinely insufficient to classify or word a specific bullet.
3. Version: read `NGINX_VERSION` from `src/core/nginx.h` at the selected current commit/tag, not from the working-tree checkout (for example, `git show <current>:src/core/nginx.h`); flag if no "Version bump" commit.
4. Date: use the selected current tag's tagger date (for example, `git for-each-ref`); sanity-check it against the corresponding `changes.xml` entry when available. If current is untagged, use `[REVIEW NEEDED]`.
5. Classify → group → output in one pass. Before returning, audit every commit in the range: each must either be represented by a bullet or intentionally skipped under Skip. For every included entry, check the commit body for "Reported by" and "Thanks to" attribution. Include explicit reporter/thanks credits for any category. For Bugfixes with no such attribution, check the `Author:` display name and email (not just the committer); if the author is outside @nginx.com/@f5.com, add `Thanks to <author>.` Do not consider an entry complete until its credit check is done. Return only the plain-text release block, without a preamble or summary.

## Skip

Docs, tests, CI, refactors, merges, version bumps, build/release infrastructure, and comment-only changes. A fix to nginx that restores building on a named supported OS or compiler, or with an already-supported dependency, is user-visible and is not build infrastructure; include it as a Bugfix. Groundwork whose only consumer is a feature introduced in this same range (the feature bullet already covers it). Do not infer that a change belongs to a separate security disclosure merely because a bot co-authored it. Exclude a security issue from public CHANGES only when the commit explicitly identifies it as undisclosed or internal; flag that exclusion for review. Changes with no user-facing surface at all (no directive/module/variable/protocol behavior an admin could observe or configure), even when framed as a "Change".

Regression note ("the bug had appeared in X.Y.Z"): add only when the commit itself flags a regression (says "regression"/"broke", or cites another commit/version). A cited bare hash needs resolving to a version: `git tag --contains <hash> --sort=v:refname | head -1`. If that hash is old (several releases back), drop the regression note; classify the change by its actual behavior rather than automatically treating it as a Feature.

## Classify

- **Security**: CVE or clear exploit. Include the CVE ID and any reporter explicitly identified in the commit.
- **Feature**: new directive/module/variable; new compatibility with a dependency version that didn't previously exist; or an old, always-missing capability finally closed. Minimal announcement.
- **Change**: behavior/default/deprecation shift with real user impact. A removed event method or option qualifies only when the commit or code establishes that it was documented, supported, or usable by configurations; do not announce removal of unfinished, inaccessible, or unsupported implementation merely because it was compiled. State the bare fact only, present tense, past participle ("has been removed", "now defaults to") — never append a rationale/justification clause (no "as it was never finished", "becauseX", "since Y"); compare changes.xml: "HTTP/2 server push support has been removed.", "obsolete aio and rtsig event methods have been removed." — nothing more.
- **Bugfix**: a user-visible behavior that was broken, including nginx failing to build on a named OS or compiler, or with an already-supported dependency. State the affected behavior or component; for example, "nginx could not be built on <OS>." A platform-specific fix is not automatically a Workaround.
- A commit explicitly fixing undefined behavior in a public protocol or module path qualifies as a Bugfix even if it does not state a concrete user-visible failure. Name the undefined behavior and its stated trigger when available; for example, "undefined behavior in HTTP/3 with empty encoder field values." Do not infer a crash or other symptom.
- **Workaround**: nginx-side mitigation for a defect in an external library, OS, or compiler, as identified in the commit body. Name the affected external component when stated; do not use this category just because a fix is platform-specific.

Ordering: put Security entries first; follow nearby release entries for the relative order of Features and Changes, which can vary; place Workarounds before Bugfixes and any grouped minor-fixes line last. Keep related entries together.

## Wording

One short sentence per bullet. Roman and Sergey keep CHANGES terse — when in doubt, cut, don't elaborate.

- Prefix each entry with its classification (`Feature`, `Change`, `Bugfix`, `Workaround`, or `Security`). Name a module or directive tersely for features, without a directive list or syntax/config snippets. Existing directives gaining a capability may state it briefly.
- For a bugfix, state the shortest concrete user-visible behavior supported by the commit, including its trigger when one is stated. Preserve named methods, directives, protocols, platform constraints, and other conditions that explain when the failure occurs; do not generalize them away. If the commit enumerates a short set of triggering operations, retain that set rather than replacing it with a broad phrase. If the commit states an observable failure or effect, name it; do not replace it with a module-only summary. Use a terse component form such as `in the ngx_http_dav_module.` only when the commit provides no more useful observable behavior. Never invent a symptom or trigger.
- Build failures and other concrete user-visible failures should name the failed operation and stated platform or trigger; do not hide them in a generic module summary.
- For crashes, hangs, security issues, data corruption, regressions, and other concrete user-visible failures, a concise symptom + trigger sentence is appropriate even when the issue is not severe. State only effects and triggers supported by the commit; preserve explicitly stated security consequences and alternatives. Do not infer an unstated fault or downstream consequence.
- Apply the symptom rule literally. For example, describe a rejected `"Priority"` field rather than merely "Priority field parsing"; say asynchronous I/O was not enabled rather than naming `FIOASYNC`; say EIO errors were logged at the alert level after SSL shutdown when using kernel TLS receive offload (do not conflate this old symptom with the fixed EOF behavior); say the request URI remained the destination URI when destination mapping failed during COPY or MOVE; retain the explicit undefined behavior and empty-field-value trigger in the HTTP/3 entry rather than reducing it to "in HTTP/3"; and state that control API error logs omitted the operating system reason when close-on-exec, nonblocking I/O, or signal-ownership setup failed, rather than saying only "socket setup failed." These examples illustrate the required specificity, not text to reuse when the commit differs.
- Every bullet must be a grammatical, complete sentence. A Workaround must say what nginx does or what observable issue occurs, not start with a fragment such as `for negative values returned by ...`; for example, `nginx now handles negative values returned by LibreSSL on memory allocation failure when serializing SSL sessions.`
- Render every credit as a separate sentence on a continuation line in the exact form `Thanks to <name>.`, after the changelog statement; do not use `Reported by`, parentheses, or lowercase `thanks to` in the output. A commit's "Reported by" and "Thanks to" text identifies the credited name; render either as `Thanks to <name>.` Use the name from an explicit attribution verbatim. Otherwise use the `Author:` display name, never infer or expand a handle from the email address. For an obvious personal name written in lowercase, capitalize the name normally (for example, `nishat shabbir` becomes `Nishat Shabbir`); preserve handle spelling and case (for example, `dgprivate` stays `dgprivate`). For Bugfixes without an explicit reporter/credit, credit the patch author when their email is outside the core team (@nginx.com/@f5.com). Do not substitute a requester, reviewer, or committer for the credited contributor. Recheck that every eligible credit is present in the final output.
- For a missing field-copy/init bug, check what the consuming code does with the missing value instead of guessing ("were not available" vs. "returned incorrect values").
- Quote alerts/log lines verbatim when the body includes one — still counts as the one symptom clause.
- Use "might" for races/timing. Use "or" for alternative effects of one issue when the commit states them, or to join symptoms from different commits that share a root cause.
- Prefer public names (HTTP/2, FastCGI, control API) over module/function names; quote directive-like values the same as directives (e.g. `"select"`). Variables get `$`, never quotes.
- Never name internal C functions/APIs (`ngx_memcpy()`, `i2d_SSL_SESSION()`, `FIOASYNC`, etc.) in the prose, even if the commit body does — describe the user-visible effect instead. The one exception is a verbatim quoted log/alert string that itself happens to contain a function name (e.g. `"ngx_slab_alloc() failed: no memory"`).
- Output plain text only: do not wrap identifiers in Markdown backticks or use Markdown formatting. Use double quotes for directive and header names, matching the existing changelog.
- Follow the English spelling used in `changes.xml` (for example, "behavior" and "initialization" rather than "behaviour" and "initialisation").
- Omit anything not stated in the body — never infer specifics.

## Grouping

Default: one bullet per distinct user-visible change or bug, not necessarily one bullet per commit. Match the concise entries in `changes.xml` and group commits that jointly implement one capability or the same behavior across modules. Keep unrelated issues separate; do not fold specific fixes into a summary paragraph.

- Same root cause across multiple commits → merge into one bullet, leading with the most concrete stated symptom.
- Parallel changes that add the same user-facing capability across modules → one Feature or Change entry naming the relevant modules/directives.
- Sibling fixes (parallel modules, same behavior and trigger) → one bullet naming both, with no unnecessary per-module detail.
- Reserve `Bugfixes in <Module>, <Module>, and <Module>.` (public component names, no symptom text) for leftover, genuinely minor fixes only — use at most once per release, only after writing individual entries for substantive fixes, and never as a substitute for a crash/hang/security/regression or concrete build-failure entry. Do not combine unrelated components into this line merely to shorten the output.
- Preserve appropriate credits from the commit body across categories; apply the bugfix reporter/author rule above without crediting a requester or reviewer as the patch author.

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
- Default to concise wording, but state a concrete symptom and trigger whenever the commit provides them and they clarify user-visible behavior.
