---
name: isimip-review
description: This skill should be used when the user asks to "review this PR", "review this diff", "review this pull request", "check this patch", or pastes/attaches a git diff and asks for feedback on bugs, code quality, or convention deviations — for ISIMIP repositories (ISI-MIP/isimip-data, isimip-qc, isimip-publisher, isimip-ea, isimip-utils) or similar Python/Django tools in the ISIMIP context (packages named `isimip_*`, code importing `isimip_utils`, or working with ISIMIP protocol definitions and NetCDF files). It performs a read-only review of the supplied diff (no running tests, linters, or builds) and reports bugs, convention deviations, and improvement suggestions against the conventions observed across the ISIMIP code base.
---

# Reviewing an ISIMIP pull request or diff

Review a diff the user provides (pasted text, a file, a local `git diff`/`git show`
output, or a PR fetched with `gh pr diff <number>`) and report findings back as text.
This is a **read-only, static review**: read the diff and, when useful, the
surrounding files it touches. Never run the test suite, `ruff`, `eslint`, `pytest`,
`npm`, type checkers, `pre-commit`, or any build/lint command as part of this skill —
the user only wants a human-style read-through.

## Scope

The skill targets the ISIMIP tool chain and code that looks like it:

- **isimip-utils** — shared library (CLI/config helpers, protocol access, patterns,
  NetCDF/xarray helpers, extractions, plots). Public API, documented with mkdocs.
- **isimip-qc** — CLI that checks (and fixes) NetCDF files against the protocol.
- **isimip-publisher** — CLI that publishes files: copies, writes JSON sidecars,
  fills the metadata database (SQLAlchemy), registers DOIs at DataCite.
- **isimip-ea** — CLI for extraction and analysis (xarray) and plotting (altair).
- **isimip-data** — Django/DRF web application (repository at data.isimip.org) with a
  React frontend, reading the metadata database written by isimip-publisher.
- Similar repositories: other `isimip_*` packages, scripts built on `isimip_utils`,
  or tools consuming the ISIMIP protocol (`protocol.isimip.org`).

Unlike a project with a written style guide, these conventions are **inferred from
the code**, and the repositories differ in details (e.g. formatting of continuation
lines, lists vs. tuples for constants). Treat them as a strong default, not as law:

- The most important reference is the **surrounding code in the touched file and
  repo**. Consistency with that beats any rule in the checklist.
- Only raise a convention deviation when it breaks local consistency, makes the code
  harder to maintain, or has a practical consequence (e.g. a config option that can
  no longer be set from `isimip.toml`). Don't nitpick differences that the repos
  themselves disagree on.
- If the diff clearly belongs to an unrelated project, say so, review it on general
  grounds only, and don't apply ISIMIP-specific points.

## 1. Get the diff

- If the user pasted a diff or pointed at a file, use that directly.
- If they reference a PR number or URL and the working directory is a clone with
  `gh` available, `git fetch`/`gh pr diff <number>` is fine to *retrieve* the diff —
  that's not the kind of "running things" being avoided; running tests/linters is.
- If there's no local clone (or `gh` isn't available), fetching the diff remotely —
  e.g. `curl` against `https://github.com/<org>/<repo>/pull/<n>.diff` or the GitHub
  API — is equally fine. When surrounding context is needed and there's no clone,
  fetch the touched files from `raw.githubusercontent.com` at the PR's head commit.
- If nothing concrete was given, ask what to review rather than guessing.

## 2. Load the conventions

- `references/checklist.md` — the fast-scan list: general Python conventions shared
  by all repos, then sections per repo type (CLI tools, isimip-qc, isimip-publisher,
  isimip-data backend and frontend), testing and packaging.
- `references/ecosystem.md` — how the repositories depend on each other and on the
  ISIMIP protocol. Load it whenever the diff touches a shared interface: anything in
  `isimip_utils`, the publisher's database models, isimip-data's `metadata` models,
  the REST API, the JSON sidecar files, protocol access, or dependency pins.

Also look at the repo's own config when a style question comes up (`pyproject.toml`
or `.ruff.toml`, `.pre-commit-config.yaml`, `eslint.config.js`) — it is the
authoritative source for what the linters enforce in *that* repo.

## 3. Review the diff

Read every changed hunk plus enough surrounding context (open the touched files if
the diff alone doesn't show it) and look for, in this order of priority:

1. **Correctness bugs** — logic errors, off-by-one (especially year ranges and
   inclusive/exclusive time slices), wrong operator, unhandled edge cases (missing
   protocol keys, empty or degenerate NetCDF axes, `None` specifiers), mutation of
   shared state (the `settings` singleton, cached properties), resources not closed,
   N+1 queries or missing `.distinct()` on M2M joins, call sites left behind after a
   rename/refactor (including templates, which fail silently).
2. **Cross-repo breakage** — changes to a shared interface that other ISIMIP repos
   rely on (see `references/ecosystem.md`): isimip-utils signatures/behaviour,
   database schema, API response shape, JSON sidecar layout, CLI option names that
   users have in `isimip.toml` or `ISIMIP_*` environment variables.
3. **Security concerns** — shell injection via `subprocess` (string commands,
   `shell=True`), path traversal from user-supplied paths in isimip-data, missing
   visibility checks for non-public/restricted datasets or caveats, secrets
   (DataCite credentials, database URLs) in code or logs, XSS in templates/JSX.
4. **Convention deviations** — see step 2 and the caveats under *Scope*. Cite the
   convention (quote or paraphrase the checklist, or point at the existing code that
   does it differently) rather than just asserting "style issue."
5. **Suggestions** — genuine improvements that aren't required: an existing
   `isimip_utils` function that could be reused instead of new code, simplification,
   better naming, a missing test for new behaviour (note it, don't write or run it),
   a missing CHANGELOG/README update for a user-visible change.

Skip nitpicks that `ruff`, `ruff format`, `eslint` or `typos` already catch
mechanically (whitespace, import order, quote style in repos that run
`ruff-format`), unless the repo doesn't run that tool and the change visibly breaks
the house style.

## 4. Report findings

Reply directly in the conversation. Structure the response as:

- **Summary** — a few sentences on what the diff actually does.
- **Important files** — a ranked list (most important first) of which files carry the
  real changes and why, e.g. "`checks/dimensions.py` — the new check" /
  "`main.py` — new CLI option wiring". Rank by how much they matter, don't just
  restate the file list in diff order. Skip only when the diff is a single file.
- **Bugs** — file:line, what's wrong, why it's wrong, concrete fix. Cross-repo
  breakage and security issues go here too, marked as such.
- **Convention deviations** — file:line, the convention it departs from, concrete fix.
- **Suggestions** — file:line, what could be better, why.
- Skip a section entirely if it has nothing in it (except Summary, which is always
  present); don't pad with "no issues found" filler.

Keep each finding to a few sentences: point at the exact location, state the
problem, propose the fix. Prefer a short suggested snippet over a long explanation
when a code change is clearer than prose.

## 5. Local HTML report (Claude Code only)

After replying with the text findings, also write a small self-contained HTML
version of the same report to disk — but **only when running in Claude Code (the
CLI)**, never in the claude.ai web interface.

Always write the report into the current working directory (the workspace), never
into a scratchpad or temp directory, so the user can find it next to their
work. Name it after the reviewed target, e.g. `rdmo-pr-<number>-review.html` for a
PR, or `rdmo-diff-review.html` for a pasted/local diff.

## 6. Before ending the turn

If step 5 applied (Claude Code, Write/Bash available), confirm out loud in your
final message — one line is enough — that the HTML report was written, and give
its path. A review that only produced the text reply is incomplete; don't let a
text-only response read as "done" when the HTML file was never written. If you
skipped step 5 because this was genuinely the claude.ai web interface, that's
correct — no report, no mention of one.
