# Explorer briefs

Recon agents answer questions so the conductor never opens a file. Launch via the Agent tool with `subagent_type: "Explore"` and always set `model` explicitly — `"sonnet"` (Sonnet 5) or `"opus"` (Opus 4.8); never omit it to inherit the session model. Recon may also go to a **Codex** agent as a read-only run per [codex.md](codex.md) when that fits better. State the search breadth explicitly: `"medium"` for a targeted question, `"very thorough"` when the answer may live in several places or under unknown naming.

Explore agents are read-only locators — they find and summarize, they do not judge quality. A "is this code any good" question is a reviewer brief, not an explorer brief.

## Brief template

```
QUESTION: <the one thing you need answered>
WHY: <one sentence — what decision this feeds>
SCOPE: <dirs/areas to search, or "whole repo"> — breadth: <medium | very thorough>
Return ONLY the contract block below.
```

## Output contract

```
ANSWER: <direct answer, ≤3 sentences — for enumeration questions, one line per item instead>
POINTERS: <file:line — what it shows, one per line>
GAPS: <what could not be determined — omit if none>
```
