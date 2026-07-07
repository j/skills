---
name: enhance-spec
description: Enhance an existing spec file with typed contracts, call stacks, and a test plan — output inline, or written where told.
disable-model-invocation: true
argument-hint: [path-to-spec] [optionally, where to write the output]
---

# Enhance Spec

Enhance an existing spec by appending a **typed call-stack architecture handoff**: code-shaped contracts plus execution flows. Prefer TypeScript pseudocode over prose wherever precision matters.

This skill is design-only. Do not implement. The deliverable is a `## Technical Design` section delivered to the output target.

## Input

The invocation is natural language. It names the spec file to enhance and, optionally, where to deliver the output. Read the spec first. If no spec path was given or the file does not exist, ask for the path before doing anything else.

The **output target** is where the enhancement lands, read from the invocation's intent:

- A destination file was named ("save to…", "append to…", a second path): append to it if it exists, create it if it does not. Do not modify the spec file.
- Told to update the spec itself: append to the spec file.
- No destination named: output the enhancement directly in the reply. Do not modify any file.

Completion criterion: the target spec has been read in full and the output target is known before any other step.

## Branch selection

1. Use **Path A: Enhance directly** when the spec, conversation, docs, or codebase already contain enough background to define contracts and call stacks.
2. Use **Path B: Grill first** when the spec leaves out the problem, constraints, design direction, affected code, or acceptance criteria needed to write contracts.

If a question can be answered by exploring the codebase, inspect the codebase instead of asking.

Completion criterion: the branch is chosen from actual available context; missing architectural decisions are not invented.

## Path A: Enhance directly

### 1. Load standards and local context

Read: Project root's `CODING_STANDARDS.md`, `AGENTS.md`, or `CLAUDE.md` for coding standards.

Inspect existing code/docs for local vocabulary, module layout, domain concepts, error handling, adapters, observability, runtime patterns, and test style.

Completion criterion: the enhancement uses project vocabulary and does not introduce a pattern, library, adapter, schema style, or test strategy before checking local precedent.

### 2. Extract the design problem from the spec

From the spec (supplemented by code and docs), capture:

- current state;
- problem;
- users/callers;
- goals;
- non-goals;
- constraints;
- invariants;
- affected systems;
- likely entrypoints;
- operational/runtime concerns;
- risks;
- open questions.

Mark unknowns as open questions instead of filling gaps with plausible design. If the codebase contradicts the spec, flag the contradiction as an open question — do not silently override the spec.

Completion criterion: every claimed requirement or constraint is grounded in the spec, conversation, code, docs, or an explicit open question.

### 3. Explore design alternatives (only if the spec leaves design open)

If the spec already fixes the design, adopt it and skip to step 4.

Otherwise, produce materially different alternatives before choosing the recommended design. Alternatives should differ in interface shape, seam placement, ownership, call stack, runtime topology, or module boundaries — not just names.

For each alternative, sketch:

- domain types and state model;
- public/module interfaces and APIs;
- input/output types;
- expected failure types;
- seams, boundaries, and adapters;
- entrypoint-to-side-effect call stack;
- parsing/projection strategy;
- authorization, observability, cancellation, idempotency, and transaction flow when reachable;
- test seam strategy;
- tradeoffs.

Compare alternatives on:

- caller burden;
- module depth and leverage;
- locality of invariants and change;
- seam placement;
- boundary parsing and projections;
- error and cancellation model;
- testability through real seams;
- operational/runtime fit;
- implementation complexity.

Completion criterion: the design is either adopted from the spec, or the recommendation is chosen after comparing alternatives — not before.

### 4. Specify the typed contracts

For the design, outline every new, changed, or deleted:

- domain value;
- branded/refined type;
- state machine variant;
- input/output type;
- request/response shape;
- function signature;
- class or module interface;
- expected-failure/custom-error type;
- adapter interface;
- protocol DTO;
- persistence DTO/projection;
- runtime-boundary codec;
- public API.

Name seams, adapters, implementations, ownership boundaries, and what crosses each boundary. State what each layer may know and what must not leak across the seam.

Completion criterion: every new or changed boundary has a concrete type/interface/API sketch, or an explicit reason no new contract is needed.

### 5. Specify call stacks and data flow

For every new, changed, or deleted behavior, show the call stack from entrypoint to side effects and response.

Include type/data flow:

```txt
raw input
  -> boundary DTO / unknown
  -> parser
  -> canonical domain/application input
  -> service/module interface
  -> adapter call
  -> typed result/error
  -> projection
  -> serialized output
```

Include current vs proposed flow when changing existing behavior. Include failure, retry, cancellation, transactionality, idempotency, observability, authorization, and runtime-hop flow when reachable.

Completion criterion: every affected behavior has an end-to-end call stack and type/data-flow trace.

### 6. Map files and modules

List:

- files/modules to add;
- files/modules to change;
- files/modules to delete, if any;
- test files;
- config/migration/runtime files, if any.

For each file, state the contract, code path, boundary, adapter, domain concept, or test responsibility it owns.

Completion criterion: every contract and call-stack step maps to a file/module or an open question.

### 7. Write the RGR TDD test plan

Follow the `/tdd` / `$tdd` skill (when installed) and the project's testing standards. Plan vertical Red-Green-Refactor slices: one failing behavior test, minimal implementation, repeat. Do not write a horizontal "all tests first, all code later" plan.

Favor behavior through public interfaces and real seams over implementation-coupled mocks.

Cover proportionately:

- happy paths;
- failure paths;
- parser rejection and accepted shapes;
- domain invariants and state transitions;
- adapter contracts;
- persistence/runtime semantics;
- cancellation/retry/idempotency paths;
- observability and safe summaries where relevant;
- end-to-end flows for high-consequence behavior.

Completion criterion: every public behavior, invariant, important failure path, changed boundary, and changed seam has a red test slice or an explicit reason not to test it.

### 8. Deliver the enhancement to the output target

Deliver the enhancement section (outline below) to the output target — appended to the end when the target is an existing file. Do not rewrite, reorder, or restate the spec's existing content — reference it. If the spec already has a section the enhancement would duplicate (e.g. a file map), extend or point to it instead of repeating it. When the output target is a separate file, open the enhancement with a link back to the spec file.

Do not implement and do not ask to implement by default.

Completion criterion: the enhancement follows the outline below at the output target, and the result is implementation-ready for another engineer.

## Path B: Grill first

1. Do not enhance yet.
   - State that the spec does not carry enough context for an implementation-ready enhancement, and what is missing.
   - Completion criterion: the agent has not invented requirements, APIs, files, or call stacks.
2. Start a grilling interview.
   - Use the `/grill-with-docs` / `$grill-with-docs` skill (when installed) when the user wants docs, ADRs, glossary/domain language, or durable design artifacts created during discovery.
   - Otherwise use the `/grill-me` / `$grill-me` skill (when installed). With neither installed, run the interview directly by the rules below.
   - Ask one question at a time and provide the recommended answer with each question.
   - If a question can be answered by exploring the codebase, inspect the codebase instead of asking.
   - Completion criterion: the interview has enough context for Path A: problem, users/callers, constraints, affected systems, desired behavior, boundaries, likely APIs, invariants, risks, and acceptance tests.
3. Convert to the enhancement.
   - Once grilling context is sufficient, run Path A.
   - Completion criterion: the final artifact is a typed call-stack architecture handoff written to the output target, not interview notes.

## Required enhancement outline

Deliver this shape to the output target, unless the task is tiny enough to compress without losing contracts or call stacks:

```md
## Technical Design

### Design Overview

### Alternatives Considered <!-- only if the spec left design open -->

#### Option 1: <name>

#### Option 2: <name>

#### Recommendation

### Domain Model and Types

### Types, Interfaces, and APIs

### Seams, Boundaries, Adapters, and Implementations

### Call Stacks and Data Flow

#### Current / Old Flow

#### Proposed / New Flow

#### Failure Flow

#### Retry / Cancellation / Idempotency Flow

#### Observability Flow

### Files to Add / Change / Delete

### RGR TDD Test Plan

### Risks and Open Questions
```

Omit sections that truly do not apply, but do not omit typed contracts, seams, call stacks, or tests merely because they are hard to specify. When appending to an existing file, nest the headings one level deeper or shallower to fit its heading structure.

## Writing rules

- Code first: TypeScript pseudocode defines contracts, APIs, and data flow.
- Prose explains why; types and call stacks define what changes.
- Focus on types, interfaces, APIs, inputs/outputs, seams, boundaries, adapters, domain modules, service modules, external adapters, and call stacks.
- Prefer precise domain values over strings, booleans, nullable bags, and loosely shaped objects.
- Keep seams real: adapters translate framework, persistence, network, time, randomness, telemetry, runtime, or platform boundaries.
- Avoid speculative abstraction; every seam earns its existence through invariants, locality, leverage, testing, or a real boundary.
- Keep a single source of truth; the enhancement references the spec's existing sections instead of restating them, and does not restate its own rules across sections.
- Unknowns stay open questions. Do not invent product requirements, domain rules, APIs, or call stacks to make the spec feel complete.
