# Reviewer briefs

## The reviewer pair

The initial review is **two sub-agents launched concurrently** (one message, two Agent calls), each with its own lane:

- **Correctness reviewer** — judges the diff against the implementer's INTENT and DONE MEANS: bugs, broken behaviour, missing verification, unmet criteria.
- **Standards reviewer** — judges only guideline compliance: naming, structure, idiom, conventions. Its standards come from the repo's AGENTS.md / CLAUDE.md and a `CODING_STANDARDS.md` at the repository root — if the brief doesn't already supply the standards, its first action is to read those files (skip any that don't exist) before judging anything.

Either reviewer may load AGENTS.md / CLAUDE.md / CODING_STANDARDS.md at any point — the standards files are not the standards lane's private property; the correctness reviewer should consult them when a finding hinges on a convention.

Pick models from the decision matrix; the two lanes may use different models. Launch options per lane:

- **Claude lane (opus-4.8 / fable-5):** launch via the Agent tool (`model: "opus"` or `"fable"`).
- **gpt-5.5 lane:** launch a read-only `codex:codex-rescue` run carrying the same brief — see [codex.md](codex.md) for the launch and prompt-wrapping rules.

For high-stakes changes, run one lane on gpt-5.5 so the pair spans two model families, and compare verdicts.

Merge the two contracts, dedupe overlapping findings (keep the higher priority), and adjudicate once — a single fix turn carries the merged standing findings. Both verdicts are advisory: the conductor rules, and may dismiss any finding or overrule an `approve` and order fixes anyway. Never auto-forward findings to the implementer.

Re-reviews (when the task loop calls for one) are a **single** agent, not the pair — briefed only to confirm the named standing findings were fixed, not a fresh full pass.

## Brief templates

Shared body for both lanes:

```
REVIEW: <branch / commit range / files changed — never "the recent changes">
ISSUE: <issue reference — omit line if none>
INTENT: <the implementer's TASK and DONE MEANS, verbatim>
Judge only what is there — do not propose redesigns or fix anything yourself. Read-only: change no files, with one exception — if ISSUE is set, record your findings on it in the exact sub-task format from the reviewer reference, via the Issue Tracker described in the repo's AGENTS.md / CLAUDE.md. Nothing else goes on the issue.
Return ONLY the contract block below.
```

Prepend the lane header:

**Correctness lane** —

```
Load the code-review skill (`Skill(code-review)`) and run it against the diff below before judging anything; map its findings into the contract. (Claude lane only)
FOCUS: correctness — bugs, broken behaviour, unmet DONE MEANS. Leave pure style/convention issues to the standards reviewer, but consult AGENTS.md / CLAUDE.md / CODING_STANDARDS.md whenever a finding hinges on a repo convention.
```

**Standards lane** —

```
FOCUS: coding standards only — do not re-litigate correctness.
STANDARDS: <paste the known standards, or:> not supplied — read the repository's AGENTS.md / CLAUDE.md and the root CODING_STANDARDS.md first (skip any that don't exist) and review against them; flag any violation as a finding. If none of these files exist, judge against the dominant conventions of the surrounding code.
```

## Output contract

```
VERDICT: approve | fix
FINDINGS: <one per line: [P1|P2|P3] file:line — defect + concrete failure scenario, ≤1 line>
NOTES: <≤2 bullets of non-blocking observations — omit if none>
```

P1 = breaks correctness or the brief's DONE MEANS; P2 = violates repo guidelines; P3 = polish. An `approve` verdict must have no P1/P2 findings.

## Issue sub-task format

When the brief carries an `ISSUE:` line, each reviewer appends exactly one section to that issue — sub-tasks the implementer will later mark completed or skipped. Minimal by design: one checkbox per finding, ordered P1 → P3, no prose, no restating the diff or the verdict reasoning.

```
## Review <n> — <lane> — <approve|fix>
- [ ] **P1** `path/to/file.ts:42` — <defect, ≤1 line>
- [ ] **P2** `path/to/file.ts:88` — <defect, ≤1 line>
```

`<lane>` is `correctness` or `standards`; both lanes of the same round share the same `<n>` (given in the brief by the conductor, starting at 1). An `approve` with no findings adds only the heading line. If the other lane already flagged the same defect on the issue, skip the duplicate. The implementer's fix turn closes each item by checking the box and appending `→ fixed` or `→ skipped: <ruling>` — the reviewer never checks boxes itself.
