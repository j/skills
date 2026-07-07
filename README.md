# skills

My personal collection of Agent Skills.

## Skills in this repository

### [conductor](skills/conductor/SKILL.md)

Runs a Claude Code session as a **conductor**: the main session never reads, searches, or edits code itself. Every piece of work — recon, planning, implementation, review, fixes, commits — is delegated to a sub-agent chosen from a decision matrix, and the conductor acts as the final judge over every agent's output. Keeping only decisions, task state, and compact agent reports in context is what keeps a long session sharp.

Key ideas:

- **Decision matrix** — maps each kind of work to an agent and model: Explore agents for recon, Plan agents for design, Codex (gpt-5.5) for clear-spec implementation, Opus for user-facing work, diagnosticians for QA and debugging, and Fable as the escalation model when a default model misses the bar.
- **Task loop** — one task = one implementation agent = one commit: decompose → recon → shape gate (a design contest, run only when no existing spec pins the shape) → brief and implement → review gate → adjudicate → (capped) re-review → commit. Multi-task runs close with a single session review over the whole commit range.
- **Review tiers** — the review gate is weighed to the blast radius: a verified mechanical change commits on the implementer's contract alone, a small contained task gets a single correctness reviewer, and everything else gets two concurrent reviewers (a correctness lane and a standards lane) — spanning two model families for high-stakes work. The conductor rules on which findings stand.
- **Bounded loops, work to completion** — every review cycle is hard-capped at two rounds; at the cap the conductor rules and the task commits. Agents never spin: tasks run to completion, and work only stops for a genuine blocker needing human input — after a fable-5 escalation has failed to clear it — with immediate notification.
- **Peer briefs** — every sub-agent gets an explicit brief (goal, constraints, in-scope files, done-criteria) and must return a compact output contract, nothing more.

Reference files: [explorer](skills/conductor/references/explorer.md) · [planner](skills/conductor/references/planner.md) · [tech-spec](skills/conductor/references/tech-spec.md) · [implementer](skills/conductor/references/implementer.md) · [reviewer](skills/conductor/references/reviewer.md) · [diagnostician](skills/conductor/references/diagnostician.md) · [codex](skills/conductor/references/codex.md)

### [enhance-spec](skills/enhance-spec/SKILL.md)

Enhances an existing spec file by appending a **typed call-stack architecture handoff**: TypeScript-pseudocode contracts, seams and adapters, entrypoint-to-side-effect call stacks, a file map, and a red-green-refactor test plan. Design-only — output goes inline or to a target file, never into implementation.

Invoked directly as `/enhance-spec <spec-path> [output-target]`, and it powers the conductor's tech-spec contest: a scribe agent writes the base spec from the conductor's brief, both contest designers apply this skill's method to it, and the judge writes the winning `chosen.md` in its outline before appending the design to the task's issue. Install it alongside the conductor skill.

## External tools & skills to install

The conductor skill delegates to tools and skills that are not part of this repository:

- **[Codex CLI](https://github.com/openai/codex)** — runs the gpt-5.5 implementation and review agents; the conductor drives it directly via `codex exec` (no plugin), per the [codex reference](skills/conductor/references/codex.md). Install with `npm install -g @openai/codex`, authenticate with `codex login`, then run the reference's once-per-machine sandbox probe.

Many of the skills referenced are available in **[mattpocock/skills](https://github.com/mattpocock/skills)**. Skills from that repository used by the skills here:

- [tdd](https://github.com/mattpocock/skills/tree/main/skills/engineering/tdd) — implementer agents are briefed to use `/tdd` (when installed) at the seams agreed in the plan.
- [code-review](https://github.com/mattpocock/skills/tree/main/skills/engineering/code-review) — the correctness reviewer lane loads the `code-review` skill before judging a diff.
- [diagnosing-bugs](https://github.com/mattpocock/skills/tree/main/skills/engineering/diagnosing-bugs) — diagnostician agents load `diagnosing-bugs` (when installed) before rooting out a bug's cause.
- [grill-me](https://github.com/mattpocock/skills/tree/main/skills/productivity/grill-me) and [grill-with-docs](https://github.com/mattpocock/skills/tree/main/skills/engineering/grill-with-docs) — enhance-spec's Path B interview runs on these (when installed); without them it interviews directly.
