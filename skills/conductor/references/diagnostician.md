# Diagnostician briefs

The diagnostician is the conductor's QA and debugging lane: it verifies that behaviour is real, roots out the cause of a bug, and figures out why a stuck agent is stuck. It diagnoses — it never fixes. A diagnosis in hand becomes a normal implementer brief; the findings are the diagnostician's only product. It is not a locator — "where does X live" stays an explorer brief.

Models: **Codex** (a read-only run per [codex.md](codex.md)) or **opus-4.8** (Agent tool, `subagent_type: "claude"`, `model: "opus"`). Depth can wander: if the diagnostician sprawls — chasing side quests, piling up theories, making a mess of its report — disregard its work entirely and launch a fresh agent on **fable-5** with the same brief. Replace a wanderer; never iterate with one.

## When to launch

- **QA** — prove a feature actually behaves as claimed, beyond what the review gate can judge from a diff.
- **Bug diagnosis** — a defect with an unknown cause. The diagnostician returns the root cause; the fix goes to an implementer per the matrix.
- **Stuck agent** — *any* agent that stalls, loops, or returns `blocked` or repeated `partial` gets a diagnostician: hand it the stuck agent's brief and its contract returns, and ask why. The diagnosis feeds your next move — re-brief, relaunch, or escalate to fable-5; the retry budget is the task loop's rule, not yours to extend.

## Brief template

```
DIAGNOSE: <the symptom or claim — what is wrong, or what must be proven to work>
WHY: <one sentence — what decision the diagnosis feeds>
ISSUE: <issue reference — omit line if none>
CONTEXT: <repro steps, error output, file:line pointers from recon; for a stuck agent: its brief and contract returns, verbatim>
SCOPE: <where to look; what is out of bounds>
If this is a bug/debugging task: load the `/diagnosing-bugs` / `$diagnosing-bugs` skill first, if available, and follow it.
Read-only: change no files. Diagnose, don't fix.
Return ONLY the contract block below — the contract is the sole channel back to the main thread; any finding outside it is lost.
```

## Output contract

The brief must quote this contract verbatim — the diagnostician's return is decision data for the conductor, and findings that don't land in these fields don't make it back.

```
DIAGNOSIS: <root cause in ≤3 sentences — the mechanism, not the symptom>
EVIDENCE: <one line per claim: file:line or the command run + what it showed>
FIX DIRECTION: <≤2 bullets — what the implementer brief should say; not a patch. For a stuck agent: how to re-brief or whether to relaunch>
CONFIDENCE: high | medium | low — <if not high: the one check that would confirm it>
GAPS: <what could not be determined — omit if none>
```

For QA runs, `DIAGNOSIS` is the verdict — "behaves as claimed" with `EVIDENCE` of what was exercised, or the defect found.
