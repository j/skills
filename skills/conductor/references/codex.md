# Codex (GPT-5.5) agents

The conductor runs GPT-5.x coding agents through the Codex CLI directly: `codex exec`, no plugin. The conductor never runs `codex` itself — a brief is its only medium. Every Codex run is carried by a **runner**: a Claude sub-agent (Agent tool, `subagent_type: "claude"`, `model: "sonnet"`) whose brief holds the exact command line. The runner executes it via Bash, extracts the thread id from the first line of the JSONL event stream, verifies the result with git, and relays Codex's final message (the `-o` file) verbatim as the contract. Runners are disposable; the durable handle is the **thread_id, not the agent**.

Prerequisites: the Codex CLI (`npm install -g @openai/codex`), authed (`codex login`). `codex doctor` health-checks the install and auth; `codex sandbox -- <cmd>` pre-flights "would codex be allowed to do X?" without a model call. Probe once per machine — a runner runs `codex doctor` and `codex sandbox -- echo ok`. If the sandbox backend is broken (e.g. nested containers without the landlock LSM), route Codex-lane work to Claude lanes or escalate per the ladder in §9, with its process discipline.

## TL;DR cheat sheet

```bash
# One-shot delegated task (the canonical invocation)
codex exec --json \
  -m gpt-5.5 -c model_reasoning_effort="high" \
  --sandbox workspace-write --strict-config \
  -C /abs/path/repo \
  -o /abs/path/last-msg.txt \
  "<task>" > /abs/path/events.jsonl 2>/abs/path/stderr.log </dev/null

# Session id (resume/steering handle) = first line of events.jsonl:
#   {"type":"thread.started","thread_id":"019f3d8e-..."}

# Steer / continue (--sandbox is INVALID on resume; use -c sandbox_mode)
codex exec resume <THREAD_ID> --json \
  -c sandbox_mode="workspace-write" -c model_reasoning_effort="high" \
  -o /abs/path/last-msg.txt "<new instruction>" \
  > /abs/path/events2.jsonl 2>>/abs/path/stderr.log </dev/null

# Interrupt a running task (SIGINT preferred; SIGKILL equally resume-safe)
kill -INT <codex-pid>    # then resume with corrective instructions

# Machine-readable code review
codex exec review --base main --json -o /abs/path/findings.txt \
  -c model_reasoning_effort="medium" > /abs/path/review-events.jsonl 2>/dev/null </dev/null
```

**Golden rules**
1. Exit code 0 means "the run completed", **not** "the task succeeded". Detect success from the
   final `agent_message` (or `--output-schema` with a `status` field).
2. Always pass `--sandbox workspace-write` explicitly for coding tasks (the default depends on
   mutable per-directory trust records).
3. Always pass `--strict-config` (typo'd `-c` keys are otherwise silently ignored).
4. Always redirect `</dev/null` and `> file` — never let codex stream into an agent's context.
5. To steer, **kill first, then `codex exec resume`**. A concurrent resume against a live session
   silently forks a stale snapshot (no lock, no error).
6. No PTY is needed anywhere. `codex exec` is genuinely non-interactive.

---

## 1. Model and reasoning effort

Use `gpt-5.5`. Reasoning efforts: `low | medium | high | xhigh` (default `medium`).

- Set with `-m gpt-5.5 -c model_reasoning_effort="high"` — works on `exec`, `exec review`, and
  `exec resume`.
- Higher effort is slower. Pick the lowest effort that fits the task.
- The global `~/.codex/config.toml` pins `xhigh` — pass `-c model_reasoning_effort=...` on
  **every** invocation or you silently pay xhigh latency.
- `-c model_verbosity="low|medium|high"` controls output verbosity.
- Prefer explicit `-c` overrides over profiles (`-p name`): a nonexistent profile name is
  silently ignored (exit 0, base config used).

| Task | Effort |
|---|---|
| Mechanical edits, small scoped fixes | `low` |
| Normal feature work | `medium`–`high` |
| Hard debugging, architecture, second opinion | `high`–`xhigh` |
| Code review sweep | `low` (fast) or `medium` (better recall) |
| **Conductor lane:** recon ([explorer.md](explorer.md)) | `low`–`medium` |
| **Conductor lane:** contest designer ([tech-spec.md](tech-spec.md)) | `medium` |
| **Conductor lane:** implementer + fix turns ([implementer.md](implementer.md)) | `medium`–`high`; mechanical asks `low` |
| **Conductor lane:** review gate ([reviewer.md](reviewer.md)) | `medium` |
| **Conductor lane:** diagnostician — QA, bug diagnosis, unstick ([diagnostician.md](diagnostician.md)) | `xhigh` |

This table is the **only** place model and effort are picked — the decision matrix and lane files just say "Codex". The runner's command always passes `-m gpt-5.5` and the effort explicitly (never rely on the global config), and never raise effort to compensate for a vague brief; a better contract beats "think harder".

---

## 2. Invocation surfaces

| Surface | What it is | Verdict |
|---|---|---|
| `codex exec` + `codex exec resume` | Non-interactive turns; JSONL events; disk-persisted sessions | **Default. Use this.** |
| `codex exec review` | Non-interactive code review with `--json`/`-o`/`-m` | **Use for reviews** (top-level `codex review` lacks those flags) |
| `codex mcp-server` | MCP stdio server (`codex` / `codex-reply` tools) | Skip — adds nothing over `exec --json` + `resume` |
| `codex app-server` | Experimental JSON-RPC server with live mid-turn steering (`turn/steer`) | Skip unless kill+resume proves inadequate |
| `codex` (TUI) | Interactive terminal app | Never from agents |
| `codex sandbox -- <cmd>` | Run any command under codex's sandbox, no model | Pre-flight: "would codex be allowed to do X?" |

The sandbox is Landlock + seccomp and works when codex is spawned from an agent context;
`--sandbox danger-full-access` is not needed for normal work. `codex doctor` health-checks the
install and auth.

---

## 3. One-shot invocation flags

```bash
codex exec --json -m gpt-5.5 -c model_reasoning_effort="high" \
  --sandbox workspace-write --strict-config -C /abs/path/repo \
  -o last-msg.txt "<task>" > events.jsonl 2>stderr.log </dev/null
```

- `--json` → stdout is a compact JSONL event stream (see Appendix A).
- Without `--json`: stdout = final assistant message **only**; stderr = full human transcript.
  So `$(codex exec ... 2>/dev/null)` is clean.
- `-o last-msg.txt` → exactly the final assistant message, no noise. The cheapest thing for a
  monitor to read.
- `--output-schema schema.json` → final message is schema-conforming JSON. **Every**
  `agent_message` becomes schema-constrained, including in-progress ones — always take the LAST
  one (or read the `-o` file). Recommended schema for delegation:
  `{status: "completed"|"blocked"|..., files_changed: [...], summary: ...}` — this is how you
  detect task success, since exit codes can't.
- `--ephemeral` → nothing persisted, **no resume possible**. Throwaway queries only.
- `--skip-git-repo-check` → run outside a git repo (codex otherwise refuses).
- `-C <dir>` / `--cd <dir>` → operate on the target repo regardless of cwd.
  `--add-dir <dir>` grants extra writable roots.
- Prompt via stdin: `codex exec ... - < prompt.md` — avoids shell-quoting hell for long prompts.
  When both stdin and an argument are given, stdin is appended as a `<stdin>` block.
- `codex exec` has no approval prompts — blocked actions are reported back to the model. This is
  why explicit `--sandbox` matters.

Conductor note: the lanes speak the **text contract** — the lane's output contract embedded in `<compact_output_contract>` (§11), relayed verbatim from the `-o` file. Reach for `--output-schema` only when a return must be machine-parseable (a verdict a script consumes, a data-extraction lane).

---

## 4. Wrapper-agent pattern (runners)

Run codex from a wrapper subagent, with all codex output redirected to files:

- A real codex task produces tens of KB of JSONL and transcript. A subagent absorbs it; the
  conductor receives only the subagent's final message.
- The wrapper's job is mechanical (invoke, watch files, parse JSONL, distill) — a small, cheap
  model is plenty.
- **The wrapper is disposable.** Codex session state lives on disk (see Appendix B), so any
  later agent can steer or continue via `codex exec resume <thread_id>`. The durable handle is
  the **thread_id, not the agent**.

### The runner brief (conductor lanes)

The conductor's runner brief carries the brief text and the exact command line, plus lane-specific verification:

```
RUN CODEX: Write the text between the BRIEF markers to <SCRATCH>/brief.md verbatim, then via Bash
execute the command between the CMD markers exactly as written — no flag changes, no reformatting
(timeout 600000; run_in_background if it may exceed 10 min, then poll last-msg.txt / the last
JSONL event — never cat the full events.jsonl).
<<<BRIEF
<the XML-wrapped brief, per §11>
BRIEF>>>
<<<CMD
codex exec --json -m gpt-5.5 -c model_reasoning_effort="<effort>" \
  --sandbox workspace-write --strict-config -C <ABS-REPO> \
  -o <SCRATCH>/last-msg.txt - < <SCRATCH>/brief.md \
  > <SCRATCH>/events.jsonl 2><SCRATCH>/stderr.log
CMD>>>
THREAD: extract with `head -1 <SCRATCH>/events.jsonl | jq -r .thread_id` — report it even on failure.
DONE CHECK: `jq -rs 'any(.[]; .type=="turn.completed")' <SCRATCH>/events.jsonl` must be true —
false means the run was killed or is still running. Exit 0 means "the run completed", not "the
task succeeded": judge success from the -o file.
VERIFY: <lane- and turn-specific git evidence — coding turn: `git -C <ABS-REPO> status --porcelain`
lists only expected files; commit turn: `git -C <ABS-REPO> log --oneline -1` shows the commit and
status is clean; review lane: status is empty>
If the exit code is non-zero, the DONE CHECK fails, or stderr shows sandbox-blocked writes
(`patch rejected: writing is blocked by read-only sandbox`), return STATUS: failed plus the last
20 lines of stderr.log instead.
Return ONLY:
THREAD: <the thread id>
VERIFIED: <what the git checks showed>
CONTRACT:
<the -o file's contents, verbatim>
```

The conductor stores the returned `THREAD:` id in task state next to the task's owner — it is the fix-turn and steering handle. Swap `--sandbox workspace-write` for `--sandbox read-only` on review/recon lanes. `<ABS-REPO>` is always an absolute path; `<SCRATCH>` is the task's scratch folder. The brief goes via stdin (`-` + the file the runner just wrote) — long XML briefs never travel as a shell argument; only a short single-purpose prompt does, and then with `</dev/null`.

### Bash tool timeouts

The Bash tool's default timeout is 120000 ms (2 min) — far too short for real codex work — and
its foreground maximum is 600000 ms (10 min). Rules:

1. Always pass `timeout: 600000` on any foreground Bash call that runs codex.
2. For anything that might exceed 10 min, launch with `run_in_background` (not subject to the
   foreground cap) and poll `last-msg.txt` / the event file.
3. To raise the caps fleet-wide, set `BASH_DEFAULT_TIMEOUT_MS` / `BASH_MAX_TIMEOUT_MS` in the
   `env` block of `.claude/settings.json`.

A timeout-killed codex is fully recoverable with `codex exec resume` — treat resume as the
recovery path, not the plan.

### Steering an in-flight wrapper

`SendMessage` to it ("kill codex, resume with: <correction>"). If the wrapper already exited,
spawn a fresh agent with the thread_id. Both work because state is on disk.

Main-thread background Bash is acceptable for a single quick codex call — same command shape,
`run_in_background`, then read only `last-msg.txt`. Prefer the wrapper the moment more than one
thing is in flight. In Workflow scripts, an `agent()` stage runs the wrapper contract with a
`schema` of `{thread_id, status, summary, files_changed}` so the codex handle flows through
pipeline stages. (For the conductor the main-thread shortcut is off the table — the conductor
never runs `codex` itself; every run goes through a runner.)

---

## 5. Interrupt → steer → resume

1. **Launch** in background; record PID and `thread_id` (first JSONL line).
2. **Interrupt**: `kill -INT <pid>` — codex exits within ~2s. SIGKILL is equally resume-safe
   (rollout files are appended per-event; nothing already done is lost). Prefer SIGINT so child
   shell commands die cleanly. Target the actual `codex` binary PID — it spawns transient
   children. The event stream just ends abruptly — absence of `turn.completed` IS the
   "interrupted" signal.
3. **Steer**:
   ```bash
   codex exec resume <THREAD_ID> --json \
     -c sandbox_mode="workspace-write" -c model_reasoning_effort="high" \
     "Stop X. Instead do Y. <correction>" > events2.jsonl 2>>stderr.log </dev/null
   ```
   The same thread_id is reused (appends to the same rollout), full context including
   pre-interrupt progress survives, and codex re-verifies its prior progress against disk.
4. **Per-turn overrides on resume**: `-c model_reasoning_effort` (e.g. escalate low→high
   mid-session) and `-m` are accepted.

**Negatives:**
- `codex exec resume` **rejects `--sandbox`** (exit 2). Use `-c sandbox_mode="workspace-write"`.
- **Never resume a live session.** It succeeds (exit 0) against a stale snapshot while the live
  process runs on, and both append to the same rollout file. No lock, no error, silently wrong.
  Kill first — always.
- `codex exec resume --last` is **cwd-scoped**. Prefer explicit thread_ids when multiple
  sessions share a cwd; `--all` disables the filter. Resume also accepts thread *names*, and
  `codex fork` can branch a session.

### Fix turns — conductor notes

Fix turns go back to the implementer's thread by id, via a fresh runner:

```
cd <ABS-REPO> && codex exec resume <THREAD-ID> --json \
  -c sandbox_mode="workspace-write" -c model_reasoning_effort="<effort>" \
  -o <SCRATCH>/last-msg.txt - < <SCRATCH>/fix-brief.md \
  > <SCRATCH>/events2.jsonl 2>><SCRATCH>/stderr.log
```

- The delta is the review ruling and standing findings — never a restatement of the original
  brief. Resume targets the exact thread — deterministic, immune to interleaving: reviewer runs
  and probes launched since the implementer's run change nothing.
- **The `cd` is load-bearing.** `resume` takes no `-C`: its workdir is the invoking cwd, not the
  session's original workdir — a resume launched from the wrong directory silently runs the
  delta against the wrong repo.
- Nothing about the sandbox or effort survives resume: re-state `-c sandbox_mode=...` and
  `-c model_reasoning_effort=...` on every resume.
- Commit turns: if the lane needs Codex to commit its own work and writes under `.git` are
  blocked (EROFS on `.git/index.lock` — seen under the older bwrap backend), grant the repo's
  `.git` as an extra writable root — `--add-dir <ABS-REPO>/.git` on launch,
  `-c 'sandbox_workspace_write.writable_roots=["<ABS-REPO>/.git"]'` on resume; naming the repo's
  parent does not unprotect `.git`. Alternatively the runner itself commits after Codex finishes.
- A lost thread id is recoverable from the rollout files on disk (Appendix B) — an Explore agent
  can list them.

---

## 6. The work → review → fix loop

```
1. Wrapper agent A:   codex exec --json --sandbox workspace-write ... "<build task>"
                      → returns {thread_id, status, summary, diff_stat}
2. Reviewer agent B:  reviews the diff — either a Claude subagent reading `git diff`,
                      or codex itself: codex exec review --base main --json -o findings.txt
                      → returns findings
3. Wrapper agent C:   codex exec resume <thread_id> -c sandbox_mode="workspace-write" \
                        "A reviewer found these issues in your work: <findings>. Fix each one."
                      → codex continues WITH ITS ORIGINAL CONTEXT plus the feedback
4. Repeat 2–3 until clean.
```

A, B, C can be three different agents, or fresh spawns hours apart — the thread_id on disk is
the only continuity needed (sessions survive reboots; they're plain JSONL files). Don't use
`--ephemeral` anywhere you might want this loop.

This loop **is** the conductor's implementer-lane / review-gate / fix-turn cycle: the review gate's cap and adjudication rules from the skill apply on top of it unchanged.

---

## 7. Code review

Use `codex exec review`, not top-level `codex review` (only the `exec` variant has `--json`,
`-o`, `-m`, `--ephemeral`):

```bash
codex exec review --base main --json -o findings.txt \
  -c model_reasoning_effort="medium" > review-events.jsonl 2>/dev/null </dev/null
```

- Scope selectors: `--base <branch>` | `--uncommitted` (staged+unstaged+untracked) |
  `--commit <sha>`. Runs read-only; writes nothing to the repo.
- **Findings arrive ONLY as the final `agent_message` text** (also written to the `-o` file).
  There are no per-finding structured events, and `--output-schema` is **silently ignored** in
  review mode. The output format is stable and regex-parseable:
  ```
  <one-line verdict>

  Full review comments:

  - [P1] <imperative title> — /abs/path/file.py:START-END
    <explanation paragraph>
  ```
  Parse with `^- \[(P\d)\] (.+) — (.+):(\d+)-(\d+)$` + the following indented paragraph.
- **Custom instructions are mutually exclusive with scope flags** (`--base main "focus on X"`
  → exit 2). To scope AND instruct, put both in the prompt:
  `codex exec review "Review this branch's changes relative to main. Focus ONLY on resource
  handling and error paths."`
- `low` effort misses subtle findings (e.g. resource leaks). Use `medium`, or run 2–3 scoped
  passes in parallel wrappers and merge findings.
- Review sessions are resumable: grab `thread_id` from `thread.started` (skip `--ephemeral`),
  then `codex exec resume <id> -c sandbox_mode="workspace-write" "Fix finding #2"` — the
  reviewer becomes the fixer with full context of its own findings.
- Quirk: `turn.completed` usage counters are all zeros in review mode.

### Conductor review-lane notes

- `codex exec review` is the default Codex review lane. Its findings format is fixed (and
  `--output-schema` is ignored there), so when the conductor needs its own reviewer contract back
  verbatim, run a plain **read-only exec** instead: the reviewer brief from
  [reviewer.md](reviewer.md) wrapped in `<task>`, its output contract in
  `<compact_output_contract>`, plus `<grounding_rules>` and `<dig_deeper_nudge>` (§11), launched
  from the runner-brief template with `--sandbox read-only`.
- Name the exact diff to judge in the brief (branch, commit range, or file list).
- A read-only run cannot write the issue: strip the issue-write exception from its brief —
  findings come back contract-only, and the conductor relays them onto the issue through a
  scribe agent.
- A read-only run can't write tempfiles, so default `pytest` dies at startup with zero tests
  collected. If the brief wants the suite executed, tell the reviewer to run
  `pytest --capture=no -p no:cacheprovider`; tempfile-dependent tests still fail, so expect a
  NOTES caveat rather than a clean run.
- Every un-resumed `codex exec` is a new thread, so reviewer/implementer separation is automatic.
  The conductor's fix turns resume the *implementer's* thread; the reviewer-becomes-fixer resume
  above exists but is not the conductor's default.

---

## 8. Footguns

| # | Footgun | Mitigation |
|---|---|---|
| 1 | Exit 0 even when the task failed (e.g. sandbox blocked all writes) | Parse final `agent_message`; use `--output-schema` with a `status` field |
| 2 | Default sandbox depends on per-dir trust records **codex itself mutates** in `~/.codex/config.toml` (first run in a new dir may be read-only; later runs workspace-write) | Always explicit `--sandbox workspace-write` (exec/review) / `-c sandbox_mode=...` (resume) |
| 3 | Unknown `-c` keys silently ignored | `--strict-config` everywhere |
| 4 | Nonexistent `-p` profile silently ignored | Prefer `-c` overrides |
| 5 | `codex exec resume` rejects `--sandbox` (exit 2) | `-c sandbox_mode="workspace-write"` (or `--yolo` if full access is acceptable — see §9) |
| 6 | `--full-auto` is deprecated (still runs, warns) | Use `--sandbox workspace-write` |
| 7 | Concurrent resume of a live session = silent stale fork | Kill before steering |
| 8 | Non-TTY stdin makes codex print "Reading additional input from stdin..." and read it | `</dev/null` (or feed the prompt via `-` deliberately) |
| 9 | `--output-schema` constrains intermediate agent_messages too | Read the LAST one / the `-o` file |
| 10 | `--output-schema` ignored in review mode | Parse the stable prose format |
| 11 | Global config pins `xhigh` (slow) | Pass `-c model_reasoning_effort=...` explicitly every time |
| 12 | Review scope flags ⊻ custom prompt | Embed scope in the prompt text |
| 13 | `--ephemeral` kills resumability | Only for throwaway one-shots |
| 14 | Bash tool timeouts kill codex mid-run (default 2 min, foreground max 10 min) | See §4 timeout rules |

---

## 9. Sandbox escalation ladder

The common "sandbox issue" is footgun #2 (read-only default from trust records): stderr shows
`ERROR ... patch rejected: writing is blocked by read-only sandbox` with exit 0. That's fixed by
rung 1, not by yolo. Pre-flight any doubt with `codex sandbox -- <cmd>` and `codex doctor`.

Escalate one rung at a time:

| Rung | Flag | Effect |
|---|---|---|
| 1 | `--sandbox workspace-write` | Writes allowed in workdir + `/tmp` + `$TMPDIR`. Correct for ~all coding tasks |
| 2 | `+ --add-dir <dir>` | Extra writable roots, sandbox otherwise intact |
| 3 | `--sandbox danger-full-access` | FS/network sandbox off entirely. For broken sandbox backends (e.g. nested containers without the landlock LSM) or tasks needing system-wide writes/arbitrary network |
| 4 | `--yolo` | Alias for `--dangerously-bypass-approvals-and-sandbox`: approvals `never` **and** `danger-full-access` in one flag |

- `--yolo` is a hidden alias (absent from `--help` but accepted). Since `codex exec` never
  prompts anyway, on exec/review yolo ≈ rung 3.
- `--yolo` IS accepted by `codex exec resume` (which rejects `--sandbox`) — the one-flag escape
  hatch on resume. The precise equivalent is `-c sandbox_mode="danger-full-access"`.
- `--full-auto` is deprecated; don't use it in new automation.
- At rungs 3–4 the safety layer is process discipline, not the sandbox: clean `git status`
  before launch, narrow single-purpose prompts, dedicated workdir/worktree, `git diff` review
  before accepting, and never point it at a directory containing secrets. Prefer rung 3 over 4
  in scripts for grep-ability of intent — behavior on `exec` is the same.

Conductor lanes live at rungs 1–2 (plus `--sandbox read-only` for review and recon); rungs 3–4 only when the sandbox backend is genuinely broken on the machine, with the process discipline above.

---

## 10. Parallel fan-out

Concurrent codex instances don't interfere as long as each works in its own directory
(`resume --last` is cwd-scoped). For same-repo parallelism, use git worktrees:

```bash
git worktree add -b fix/issue-78 ../wt-78 main
# one wrapper agent per worktree, each: codex exec --json --sandbox workspace-write -C ../wt-78 ...
# conductor collects {thread_id, status, diff_stat} from each wrapper, then merges/reviews
```

---

## 11. Steer — prompting

Prompt Codex like an operator, not a collaborator: compact, block-structured, XML-tagged. A better contract beats "think harder" — never raise effort to compensate for a vague brief. One task per run; split unrelated asks.

The brief travels by stdin: the runner Writes it verbatim to `<SCRATCH>/brief.md` and the command reads it with `- < <SCRATCH>/brief.md` — nothing recomposes it en route, and shell quoting never touches it.

Codex can't load Claude skills: drop any skill-load line from a brief and state the needed instructions inline. When a skill's method matters (a designer following the `../enhance-spec/SKILL.md` / `/enhance-spec` / `$enhance-spec` skill, say), phrase it as locate-and-read — tell Codex to search for the skill folder by name, read its SKILL.md, and apply what it reads.

Wrap the conductor's brief in these blocks:

- `<task>` — the job, repo context, and the expected end state (DONE MEANS). Always.
- `<compact_output_contract>` — embed the implementer output contract here; Codex's final message comes back verbatim (the `-o` file), so this is the only verbosity control.
- `<default_follow_through_policy>` — "take the reasonable low-risk interpretation and keep going; only stop for correctness/safety/irreversible gaps." Without it Codex stops early to ask routine questions.
- Coding/fix runs: add `<completeness_contract>`, `<verification_loop>`, `<missing_context_gating>`, and `<action_safety>` (keeps changes tightly scoped, no unrelated refactors).
- Review/research runs: add `<grounding_rules>` and `<dig_deeper_nudge>`.

---

## Appendix A — JSONL event reference

```jsonc
{"type":"thread.started","thread_id":"019f3d8e-..."}        // ALWAYS first; the resume handle
{"type":"turn.started"}
{"type":"item.started",  "item":{"id":"item_0","type":"command_execution", ...}}
{"type":"item.completed","item":{"id":"item_0","type":"command_execution", ...}}
{"type":"item.completed","item":{"id":"item_1","type":"file_change", ...}}
{"type":"item.completed","item":{"id":"item_2","type":"agent_message","text":"..."}} // final answer
{"type":"turn.completed","usage":{"input_tokens":35091,"cached_input_tokens":30848,
                                  "output_tokens":199,"reasoning_output_tokens":0}}
// turn.failed appears on API errors (exit 1). An interrupted run just stops mid-stream.
```

Useful one-liners:
```bash
THREAD_ID=$(head -1 events.jsonl | jq -r .thread_id)
STATUS=$(jq -rs '[.[] | select(.item.type=="agent_message")] | last | .item.text' events.jsonl)
DONE=$(jq -rs 'any(.[]; .type=="turn.completed")' events.jsonl)   # false ⇒ still running or killed
```

If a captured stream is polluted with non-JSON lines (e.g. stderr merged into stdout, or a
missing `</dev/null`), plain `jq -s` dies on the first bad line. Sanitize first:
```bash
jq -cR 'fromjson? // empty' events.jsonl | jq -rs 'any(.[]; .type=="turn.completed")'
```

---

## Appendix B — session storage

- Rollouts: `~/.codex/sessions/YYYY/MM/DD/rollout-<timestamp>-<thread_id>.jsonl` — appended
  per-event (this is why SIGKILL is safe). Resume appends to the same file (one rollout can hold
  many turns).
- `~/.codex/session_index.jsonl` indexes interactive/named sessions; plain `exec` and review
  sessions are discovered from the sessions dir, not the index.
- `codex doctor` — health check; `codex features list` — feature flags.
