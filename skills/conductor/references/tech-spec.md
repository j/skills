# Tech-spec contest

When the task loop's shape gate finds no pinned shape, run a **contest**: two designers draft competing prototypes into scratch, a judge picks or merges the winner. No implementation code is written anywhere in the contest — every entry is a typed prototype (TypeScript pseudocode for contracts and call stacks, prose only for why).

## Designers

Launch both concurrently (one message):

- **opus-4.8 high** — Agent tool, `subagent_type: "claude"`, `model: "opus"`.
- **Codex gpt-5.5 medium** — `codex:codex-rescue` per [codex.md](codex.md), `--effort medium`, read-only except its scratch folder.

Each designer produces **two materially different alternatives** — differing in interface shape, seam placement, ownership, call stack, or module boundaries, not merely names — one file per alternative in the task's scratch folder. Designers return only paths and pitches; the concepts themselves never enter conductor context.

Brief template (both designers, same body):

```
DESIGN CONTEST ENTRY: <the task, and the one-sentence why>
ISSUE: <issue reference — omit line if none>
KNOWN: <recon facts, file:line pointers>
CONSTRAINTS: <hard requirements; follow the repo's root CODING_STANDARDS.md if it exists, else the dominant conventions of the surrounding code>
SCRATCH: write your alternatives to <task scratch folder>/<designer>-option-1.md and -2.md
Design only — write no implementation code, change nothing outside SCRATCH.
Produce TWO materially different alternatives: different interface shape, seams, ownership, call stack, or module boundaries — not the same design with different names.
Each file covers: domain types and state model; public interfaces/APIs with input/output and failure types; seams, boundaries, and adapters; the entrypoint-to-side-effect call stack; files to add/change/delete; test seams and the first red slices per the /tdd skill (vertical red-green-refactor, behaviour through real seams); tradeoffs. Ground every claim in the codebase or mark it an open question — never invent requirements.
Return ONLY the contract block below.
```

Output contract:

```
OPTIONS: <path — one-line pitch, one per line>
OPEN QUESTIONS: <what the design had to leave unresolved — omit if none>
```

## Judge

**fable-5 high** — Agent tool, `subagent_type: "claude"`, `model: "fable"`. The judge gets the four entry paths and reads them; the conductor never does.

```
JUDGE: pick or merge the strongest design from the entries below.
TASK: <the original task and constraints>
ISSUE: <issue reference — omit line if none>
ENTRIES: <the four paths with their pitches and the designers' OPEN QUESTIONS>
Criteria, in order: adherence to the repo's root CODING_STANDARDS.md (if present) and dominant conventions; elegant naming; the cleanest path by readability and code cleanliness; the design most certain to get the job done. You may merge ideas across entries.
Write the winning prototype to <task scratch folder>/chosen.md — one concrete design an implementer can follow without reading the losing entries.
If ISSUE is set: post the prototype on the issue (or its summary plus the chosen.md path) as the implementation guide, via the Issue Tracker described in the repo's AGENTS.md / CLAUDE.md.
Return ONLY the contract block below.
```

Output contract:

```
CHOSEN: <path to chosen.md>
MERGED FROM: <which entries contributed>
RULING: <why, ≤3 bullets>
OPEN QUESTIONS: <what the implementer must resolve — omit if none>
```

## After the contest

The chosen prototype is advisory like any contract — the conductor rules, and may order one revision or reverse the merge; if the revision still misses, take the strongest entry as-is and move on rather than running another round. The implementer's brief carries the `chosen.md` path in CONTEXT as the backbone of the work, along with the judge's OPEN QUESTIONS and the conductor's ruling on each — an open question the implementer never sees is a decision made by accident. A task that went through the contest normally needs no separate Plan agent: the prototype already names the files, seams, and first test slices.
