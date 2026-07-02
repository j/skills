# Planner briefs

For any task where the approach isn't obvious from the spec, launch a Plan agent (Agent tool, `subagent_type: "Plan"`, inherit model) before briefing an implementer. Feed it your recon facts — a planner that has to rediscover the codebase produces a slower, vaguer plan.

Plans for consequential work get their own review gate (models per the decision matrix) before implementation starts. Plan and review verdict alike are advisory — the conductor rules on what reaches the implementer, amending or rejecting either. The reviewed plan becomes the backbone of the implementer's brief.

## Brief template

```
TASK: <what must be built/changed, and the one-sentence why>
KNOWN: <recon facts and file:line pointers you already hold>
CONSTRAINTS: <hard requirements, things that must not change>
Return ONLY the contract block below.
```

## Output contract

```
PLAN: <numbered steps, each naming the files it touches>
RISKS: <what could go wrong, ≤3 bullets>
UNKNOWNS: <facts the plan had to assume — omit if none>
```
