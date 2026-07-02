# skills

My personal collection of Agent Skills.

## Skills in this repository

### [conductor](skills/conductor/SKILL.md)

Runs a Claude Code session as a **conductor**: the main session never reads, searches, or edits code itself. Every piece of work — recon, planning, implementation, review, fixes, commits — is delegated to a sub-agent chosen from a decision matrix, and the conductor acts as the final judge over every agent's output. Keeping only decisions, task state, and compact agent reports in context is what keeps a long session sharp.

Key ideas:

- **Decision matrix** — maps each kind of work to an agent and model: Explore agents for recon, Plan agents for design, Codex (gpt-5.5) for clear-spec implementation, Opus for user-facing work, and Fable as the escalation model when a default model misses the bar.
- **Task loop** — one task = one implementation agent = one commit: decompose → recon → brief and implement → review gate → adjudicate → (optional) re-review → commit.
- **Reviewer pair** — every coding turn is judged by two concurrent reviewers (a correctness lane and a standards lane), ideally spanning two model families; the conductor rules on which findings stand.
- **Peer briefs** — every sub-agent gets an explicit brief (goal, constraints, in-scope files, done-criteria) and must return a compact output contract, nothing more.

Reference files: [explorer](skills/conductor/references/explorer.md) · [planner](skills/conductor/references/planner.md) · [implementer](skills/conductor/references/implementer.md) · [reviewer](skills/conductor/references/reviewer.md) · [codex](skills/conductor/references/codex.md)

## External tools & skills to install

The conductor skill delegates to tools and skills that are not part of this repository:

- **[Codex CLI](https://github.com/openai/codex)** — runs the gpt-5.5 implementation and review agents. Install with `npm install -g @openai/codex`, then authenticate with `codex login`.
- **Codex plugin for Claude Code** — provides the `codex:codex-rescue` agent and the `/codex:setup`, `/codex:status`, `/codex:result`, and `/codex:cancel` commands the conductor uses to launch and manage Codex runs. Run `/codex:setup` to verify the CLI is ready.

Many of the skills referenced are available in **[mattpocock/skills](https://github.com/mattpocock/skills)**. Skills from that repository used by the conductor skill:

- [tdd](https://github.com/mattpocock/skills/tree/main/skills/engineering/tdd) — implementer agents are briefed to use `/tdd` (when installed) at the seams agreed in the plan.
- [code-review](https://github.com/mattpocock/skills/tree/main/skills/engineering/code-review) — the correctness reviewer lane loads the `code-review` skill before judging a diff.
