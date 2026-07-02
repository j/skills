# Explorer briefs

Recon agents answer questions so the conductor never opens a file. Launch via the Agent tool with `subagent_type: "Explore"`; omit `model` (inherit). State the search breadth explicitly: `"medium"` for a targeted question, `"very thorough"` when the answer may live in several places or under unknown naming.

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
ANSWER: <direct answer, ≤3 sentences>
POINTERS: <file:line per fact, one per line>
GAPS: <what could not be determined — omit if none>
```
