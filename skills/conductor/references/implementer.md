# Implementer briefs

Pick the model from the decision matrix; launch Claude implementers via the Agent tool (`subagent_type: "claude"`, `model: "opus"` or `"fable"`), Codex gpt-5.5 implementers per [codex.md](codex.md).

Keep the launch handle (agent id / Codex session id): fix turns go back to this same agent so it keeps its context.

## Brief template

```
TASK: <what to build/change, and the one-sentence why>
ISSUE: <issue reference — file path, GitHub #, Linear ID — omit line if none>
CONTEXT: <recon facts, file:line pointers, the reviewed plan or the contest's chosen.md path if one exists>
CONSTRAINTS: <hard requirements; follow the repo's AGENTS.md / CLAUDE.md conventions>
DONE MEANS: <checkable criteria — behaviour observed, tests passing, nothing left TODO>
VERIFY: <how to prove it works before reporting — run it, don't assume>
Use the /tdd skill if installed and where possible, at the pre-agreed seams: <the seams/interfaces agreed in the plan>.
While working: run typechecking regularly and the relevant single test files regularly; run the full test suite once at the end, before reporting.
If ISSUE is set: keep it current via the Issue Tracker described in the repo's AGENTS.md / CLAUDE.md — mark it in-progress when you start and ready-for-review when you return the contract. Status updates only; don't narrate progress on the issue.
Do NOT commit yet. Return ONLY the contract block below.
```

## Fix turns

After adjudicating a review, send the ruling to the same agent — SendMessage for Claude agents, a `--resume` delta for Codex (see [codex.md](codex.md)):

```
REVIEW RULING: fix findings <ids>; findings <ids> are dismissed — do not touch.
FINDINGS: <the standing findings, verbatim from the reviewer>
If ISSUE is set: the reviewers added these findings as checkboxes under `## Review <n> — <lane>` sections on the issue. As you resolve each one, check its box and append `→ fixed`; for dismissed findings, check the box and append `→ skipped: <the ruling>`. Set the issue back to ready-for-review when you return the contract.
Return ONLY the contract block.
```

If the agent is gone, brief a fresh one with the original TASK/CONSTRAINTS plus the standing findings. If a fix turn still misses the bar, relaunch the task on fable-5 with the full history of what failed.

## Commit turn

Only after the conductor approves the (re-)reviewed work:

```
APPROVED. Commit this task's work: <one-line commit message>. If ISSUE is set, mark it done. Return ONLY the contract block.
```

## Output contract

```
STATUS: done | partial | blocked
DID: <≤5 bullets, file:line each>
DECISIONS: <choices surfaced for the conductor's ruling — it may reverse any of them; omit if none>
VERIFIED: <what you ran and what it showed>
NEEDS: <blockers or questions — omit if none>
```
