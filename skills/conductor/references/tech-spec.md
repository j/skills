# Tech-spec contest

When the task loop's shape gate finds no pinned shape, run a **contest**: a scribe agent writes the base spec from the conductor's brief, two designers enhance it into competing prototypes in scratch, and a judge picks or merges the winner into one spec and appends it to the issue. No implementation code is written anywhere in the contest — every entry is a typed prototype (TypeScript pseudocode for contracts and call stacks, prose only for why).

Every design turn follows the **`/enhance-spec` / `$enhance-spec` skill**, named on the `METHOD:` line of every contest brief — each agent locates, loads, and applies the skill itself; the conductor only names it. If the skill isn't installed, drop the `METHOD:` line and brief the essentials inline instead: domain types and state model; public interfaces/APIs with input/output and failure types; seams, boundaries, and adapters; entrypoint-to-side-effect call stacks; files to add/change/delete; test seams and first red slices; tradeoffs.

## Base spec

Before designers launch, the contest needs `<task scratch folder>/spec.md` — its single problem statement, and the file every designer enhances. The conductor never writes it: the conductor composes the spec *content* inside a brief (a brief is the conductor's only medium) and a **scribe agent** puts it on disk — Agent tool, `subagent_type: "claude"`, `model: "sonnet"`:

```
SCRIBE: create <task scratch folder>/spec.md containing exactly the spec between the markers, verbatim — no edits, no additions.
<<<SPEC
<the spec content>
SPEC>>>
Change nothing else. Return ONLY: WROTE: <path>
```

The spec content is composed entirely from what is already in conductor context (the request, recon facts, prior rulings). It carries:

- problem and the one-sentence why;
- users/callers;
- goals and non-goals;
- constraints and invariants (hard requirements from the request and recon);
- known facts as file:line pointers from recon;
- acceptance criteria — what "done" means;
- open questions the conductor already knows about.

Unknowns stay listed as open questions — the spec never guesses to look complete.

## Designers

Launch both concurrently (one message):

- **opus-4.8 high** — Agent tool, `subagent_type: "claude"`, `model: "opus"`.
- **Codex gpt-5.5 medium** — `codex:codex-rescue` per [codex.md](codex.md), `--effort medium`, read-only except its scratch folder.

Each designer produces **two materially different alternatives** — differing in interface shape, seam placement, ownership, call stack, or module boundaries, not merely names — one file per alternative in the task's scratch folder. Designers return only paths and pitches; the concepts themselves never enter conductor context.

Brief template (both designers, same body):

```
DESIGN CONTEST ENTRY: <the task, and the one-sentence why>
ISSUE: <issue reference — omit line if none>
SPEC: <absolute path to the task's spec.md — your input; read it in full, never modify it>
METHOD: the `/enhance-spec` / `$enhance-spec` skill — apply its Path A end to end: load the repo's standards, extract the design problem from SPEC, then specify typed contracts, call stacks, the file map, and the RGR TDD test plan, per its required enhancement outline and writing rules.
SCRATCH: write your alternatives to <task scratch folder>/<designer>-option-1.md and -2.md
Design only — write no implementation code, change nothing outside SCRATCH.
Produce TWO materially different alternatives: different interface shape, seams, ownership, call stack, or module boundaries — not the same design with different names. Each option file is one complete, self-contained Technical Design per METHOD's outline, opening with a link back to SPEC. Two adaptations to METHOD: skip its "Alternatives Considered" section — your two files ARE the alternatives and the contest judge does the comparing — and skip its Path B entirely: you cannot interview anyone, so anything missing becomes an OPEN QUESTIONS line, never an invented requirement.
Return ONLY the contract block below.
```

Output contract:

```
OPTIONS: <path — one-line pitch, one per line>
OPEN QUESTIONS: <what the design had to leave unresolved — omit if none>
```

## Judge

**fable-5 high** — Agent tool, `subagent_type: "claude"`, `model: "fable"`. The judge gets the spec path and the four entry paths and reads them; the conductor never does.

```
JUDGE: pick or merge the strongest design from the entries below.
TASK: <the original task and constraints>
ISSUE: <issue reference — omit line if none>
SPEC: <absolute path to the task's spec.md>
METHOD: the `/enhance-spec` / `$enhance-spec` skill — your output must follow its required enhancement outline and writing rules.
ENTRIES: <the four paths with their pitches and the designers' OPEN QUESTIONS>
Read SPEC and every entry in full. Criteria, in order: adherence to the repo's root CODING_STANDARDS.md (if present) and dominant conventions; elegant naming; the cleanest path by readability and code cleanliness; the design most certain to get the job done. You may pick one entry or merge ideas across entries — the result must be ONE concrete design, not a menu.
Write the winner to <task scratch folder>/chosen.md — a complete Technical Design per METHOD's outline that an implementer can follow without reading the losing entries, opening with a link back to SPEC.
Then deliver it. If ISSUE is set, append the chosen design to the issue via the Issue Tracker described in the repo's AGENTS.md / CLAUDE.md: for a file-based tracker, append the Technical Design section to the issue file (nesting headings to fit its structure); for GitHub, Linear, or similar, post it as a comment or description update. If the tracker rejects a body that size, post the Design Overview plus the chosen.md path instead. If no ISSUE, chosen.md itself is the deliverable.
Return ONLY the contract block below.
```

Output contract:

```
CHOSEN: <path to chosen.md>
MERGED FROM: <which entries contributed>
RULING: <why, ≤3 bullets>
DELIVERED: <where the design landed on the issue — omit if no issue>
OPEN QUESTIONS: <what the implementer must resolve — omit if none>
```

## After the contest

The chosen prototype is advisory like any contract — the conductor rules, and may order one revision or reverse the merge; if the revision still misses, take the strongest entry as-is and move on rather than running another round. A revision that changes the design re-delivers to the issue the same way (the judge appends a corrected design; it never silently diverges from what the tracker shows). The implementer's brief carries the `chosen.md` path in CONTEXT as the backbone of the work, along with the judge's OPEN QUESTIONS and the conductor's ruling on each — an open question the implementer never sees is a decision made by accident. A task that went through the contest normally needs no separate Plan agent: the prototype already names the files, seams, and first test slices.
