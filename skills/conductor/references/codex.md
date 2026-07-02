# Codex (GPT-5.5) agents

The Codex plugin runs GPT-5.x coding agents in this repo through the local Codex CLI. Everything below routes through one surface: launch via the Agent tool with `subagent_type: "codex:codex-rescue"`, putting routing flags and the task in the prompt. Never invoke `Skill(codex:rescue)` from a running command — it re-enters and hangs the session.

Prerequisite: `/codex:setup` must pass (Codex CLI installed via `npm install -g @openai/codex`, authed via `codex login`).

## Create

Prompt format for the `codex:codex-rescue` agent:

```
[--background|--wait] [--resume|--fresh] [--model <id>] [--effort <none|minimal|low|medium|high|xhigh>] <the task brief>
```

- Add `--write` intent by phrasing the task as implementation — the agent defaults to write-capable runs for fix/build asks; say "read-only" explicitly for diagnosis-only runs.
- `--background` for long or multi-step tasks; manage with `/codex:status <job-id>`, `/codex:result <job-id>`, `/codex:cancel <job-id>`. Background jobs die with the Claude session that owns them.
- Codex runs with `approvalPolicy: never` — it can never pause to ask for more access, so scope the brief fully up front. Write runs are sandboxed to the workspace.

## Model

- `--model` forwards any ID verbatim to Codex; the only built-in alias is `spark` (→ `gpt-5.3-codex-spark`). Pass the gpt-5.5 ID explicitly, or leave `--model` unset and pin the default in `~/.codex/config.toml` (`model = "..."`, `model_reasoning_effort = "..."`), which is the cleanest way to make every conductor launch hit the right model.

## Manage — fix turns on the same session

Each fresh task creates a persistent named Codex thread; the thread ID is the session ID (reported in the result and via `/codex:status`). This is how the same implementer handles review fixes:

- Relaunch `codex:codex-rescue` with `--resume <delta instruction>` — it continues the latest task thread for this repo. Send only the delta (the review ruling and standing findings), not a restatement of the original brief.
- `--fresh` forces a new thread when you deliberately want a clean implementer.
- Resume fails while the thread's task is still running (`/codex:status` first) or when no prior thread exists — fall back to a fresh brief carrying the findings verbatim.

## Steer — prompting

Prompt Codex like an operator, not a collaborator: compact, block-structured, XML-tagged. A better contract beats "think harder" — never raise effort to compensate for a vague brief. One task per run; split unrelated asks.

Wrap the conductor's brief in these blocks:

- `<task>` — the job, repo context, and the expected end state (DONE MEANS). Always.
- `<compact_output_contract>` — embed the implementer output contract here; Codex's final message comes back verbatim, so this is the only verbosity control.
- `<default_follow_through_policy>` — "take the reasonable low-risk interpretation and keep going; only stop for correctness/safety/irreversible gaps." Without it Codex stops early to ask routine questions.
- Coding/fix runs: add `<completeness_contract>`, `<verification_loop>`, `<missing_context_gating>`, and `<action_safety>` (keeps changes tightly scoped, no unrelated refactors).
- Review/research runs: add `<grounding_rules>` and `<dig_deeper_nudge>`.

## Reviews via Codex

For the matrix's gpt-5.5 review lane, run the review as a read-only `codex:codex-rescue` task — there is no dedicated review command:

- Say "read-only" explicitly in the brief so the run cannot write, and always `--fresh` — a reviewer must never share a thread with the implementer it is judging.
- Wrap the reviewer brief from [reviewer.md](reviewer.md) in `<task>`, put its output contract in `<compact_output_contract>`, and add `<grounding_rules>` and `<dig_deeper_nudge>` (review-run blocks per the Steer section).
- Name the exact diff to judge in the brief (branch, commit range, or file list); use `--background` for anything beyond a couple of files.
