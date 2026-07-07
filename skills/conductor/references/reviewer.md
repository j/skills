# Reviewer briefs

The task loop's review gate sets the tier: **skip** (no reviewer — the conductor accepts on the implementer's contract), **solo** (one reviewer), or **pair** (two).

## The reviewer pair

A pair review is **two sub-agents launched concurrently** (one message, two Agent calls), each with its own lane:

- **Correctness reviewer** — judges the diff against the implementer's INTENT and DONE MEANS: bugs, broken behaviour, missing verification, unmet criteria.
- **Standards reviewer** — judges only guideline compliance: naming, structure, idiom, conventions. Its standards come from the repo's AGENTS.md / CLAUDE.md and a `CODING_STANDARDS.md` at the repository root — if the brief doesn't already supply the standards, its first action is to read those files (skip any that don't exist) before judging anything.

Either reviewer may load AGENTS.md / CLAUDE.md / CODING_STANDARDS.md at any point — the standards files are not the standards lane's private property; the correctness reviewer should consult them when a finding hinges on a convention.

Pick models from the decision matrix; the two lanes may use different models. Launch options per lane:

- **Claude lane (opus-4.8 / fable-5):** launch via the Agent tool (`model: "opus"` or `"fable"`).
- **Codex lane:** launch a read-only Codex run carrying the same brief per [codex.md](codex.md) — its conductor review-lane notes have the lane's specifics (brief wrapping, the issue-write exception, test-run caveats).

When the gate calls for a cross-family pair (high stakes), run one lane on Codex and compare verdicts.

Merge the two contracts, dedupe overlapping findings (keep the higher priority), and adjudicate once — a single fix turn carries the merged standing findings. Both verdicts are advisory: the conductor rules, and may dismiss any finding or overrule an `approve` and order fixes anyway. Never auto-forward findings to the implementer.

Re-reviews (when the task loop or session review calls for one) are a **single** agent, not the pair — briefed only to confirm the named standing findings were fixed, not a fresh full pass:

```
RE-REVIEW: confirm these findings are fixed — <the standing findings, ids and text verbatim>
REVIEW: <same scope as the initial review>
Judge only whether each named finding is resolved; flag anything a fix newly broke as a finding, but run no fresh full pass.
Return ONLY the contract block.
```

## Brief templates

Shared body for every lane, pair or solo:

```
REVIEW: <branch / commit range / files changed — paste the implementer's CHANGED list; never "the recent changes">
ISSUE: <issue reference — omit line if none>
INTENT: <the implementer's TASK and DONE MEANS, verbatim>
Judge only what is there — do not propose redesigns or fix anything yourself. Read-only: change no files, with one exception — if ISSUE is set, record your findings on it in the exact sub-task format from the reviewer reference, via the Issue Tracker described in the repo's AGENTS.md / CLAUDE.md. Nothing else goes on the issue.
Return ONLY the contract block below.
```

Prepend the lane header:

**Correctness lane** — a Claude lane opens with the skill-load line; drop it for a Codex lane (skills rule in [codex.md](codex.md)):

```
Load the `/code-review` / `$code-review` skill and run it against the diff below before judging anything; map its findings into the contract.
FOCUS: correctness — bugs, broken behaviour, unmet DONE MEANS. Leave pure style/convention issues to the standards reviewer, but consult AGENTS.md / CLAUDE.md / CODING_STANDARDS.md whenever a finding hinges on a repo convention.
```

**Solo lane** (the gate's solo tier) — same skill-load rule, ids stay `C…`:

```
FOCUS: correctness — bugs, broken behaviour, unmet DONE MEANS — plus glaring standards violations; you are the only reviewer on this task.
```

**Standards lane** —

```
FOCUS: coding standards only — do not re-litigate correctness.
STANDARDS: <paste the known standards, or:> not supplied — read the repository's AGENTS.md / CLAUDE.md and the root CODING_STANDARDS.md first (skip any that don't exist) and review against them; flag any violation as a finding. If none of these files exist, judge against the dominant conventions of the surrounding code.
```

## Session review

The closing pass of a multi-task session (matrix row "Session review after a multi-task run") is a **single** agent, not the pair — per-task gates already covered correctness and standards; this pass judges completeness and regressions across the whole run. Brief:

```
SESSION REVIEW: <full commit range of the session, first task's parent to last commit>
ISSUES: <every issue/PRD worked this session — one reference per line>
Judge exactly two things: (1) each listed issue/PRD is complete end to end — every acceptance criterion met, every review sub-task checked off as fixed or skipped with a ruling; (2) no commit in the range regressed work from an earlier commit in it. Read each issue/PRD in full before judging it.
Do not re-litigate per-task review findings already ruled on; flag only incompleteness or cross-task breakage. Read-only; fix nothing yourself.
Return ONLY the contract block.
```

Same output contract; ids `C1, C2…`. Each finding names the issue/PRD it belongs to. An `approve` here means the session is closed.

## Plan review

Reviewing a plan (matrix row "Review a plan") is a **single** agent on the shared body with two substitutions: `REVIEW:` carries the plan verbatim (there is no diff, so drop the code-review-skill line), and findings cite plan step numbers instead of file:line (ids `C1, C2…`). Same output contract. Standing findings don't get a fix turn — the conductor amends the plan itself before briefing the implementer.

## Output contract

```
VERDICT: approve | fix
FINDINGS: <one per line: <id> [P1|P2|P3] file:line — defect + concrete failure scenario, ≤1 line>
NOTES: <≤2 bullets of non-blocking observations — omit if none>
```

P1 = breaks correctness or the brief's DONE MEANS; P2 = violates repo guidelines; P3 = polish. An `approve` verdict must have no P1/P2 findings.

Ids are lane-prefixed and stable — `C1, C2…` for correctness, `S1, S2…` for standards — so the conductor's rulings, the implementer's fix turn, and the issue checkboxes all reference the same handle. When deduping merged verdicts, the surviving finding keeps its id.

## Issue sub-task format

When the brief carries an `ISSUE:` line, each reviewer appends exactly one section to that issue — sub-tasks the implementer will later mark completed or skipped. Minimal by design: one checkbox per finding, ordered P1 → P3, no prose, no restating the diff or the verdict reasoning.

```
## Review <n> — <lane> — <approve|fix>
- [ ] **C1 P1** `path/to/file.ts:42` — <defect, ≤1 line>
- [ ] **C2 P2** `path/to/file.ts:88` — <defect, ≤1 line>
```

`<lane>` is `correctness`, `standards`, or `solo`; both lanes of a pair round share the same `<n>` (given in the brief by the conductor, starting at 1). An `approve` with no findings adds only the heading line. If the other lane already flagged the same defect on the issue, skip the duplicate. The implementer's fix turn closes each item by checking the box and appending `→ fixed` or `→ skipped: <ruling>` — the reviewer never checks boxes itself. When the conductor dismisses every standing finding and no fix turn runs, a scribe agent closes the boxes with `→ skipped: <the ruling>` instead — boxes never stay dangling.
