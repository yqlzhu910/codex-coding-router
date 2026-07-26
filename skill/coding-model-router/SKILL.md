---
name: coding-model-router
description: Use when Codex needs to read, search, modify, debug, test, refactor, optimize, or architect code in a repository and the work is more than a brief explanation of short, self-contained code already pasted by the user.
---

# Coding Model Router

## Overview

Route substantive coding work to the smallest suitable surface. Keep tiny,
context-complete work in the primary agent, and preserve facts, inferences, and
unverified risks as distinct categories throughout the task.

## Classify Before Acting

| Task shape | Route |
|---|---|
| Short pasted-code explanation or tiny context-complete non-workspace task | Primary agent |
| Large read-only search, reference discovery, call-chain mapping, logs, or build errors | `code_reader` |
| Fully specified local mechanical edit or ordinary small bug | `code_worker` |
| Algorithms, architecture, cross-module work, performance, memory safety, undefined behavior, SIMD, OpenMP, concurrency, numerical methods, computational geometry, voxels, spatial indexes, or difficult debugging | `code_expert` |

Route a requested workspace modification to `code_worker` even when the exact
line is already known. Keep it in the primary agent only when the user
explicitly selects the primary agent or delegation is unavailable.

Honor explicit agent selection and read-only constraints. Do not create agents
for appearances.

## Delegate From Evidence

1. State the classification evidence.
2. Give the selected agent the goal, relevant evidence, exact scope, and
   required validation.
3. Wait for its result and independently verify material claims.

Use parallel agents only for mutually independent read-only work. A single
`code_reader` is the default; add another reader only when the searches can be
cleanly partitioned and parallel work materially helps.

## Escalate Without Losing Evidence

- When a reader finds complex reasoning or modification is required, preserve
  its evidence and escalate the work to `code_expert`.
- When a worker reaches difficult or unclear scope, stop the worker and
  escalate its evidence and exact working-tree changes to `code_expert`.
- Never hand complex implementation from `code_expert` back to `code_worker`.

## Serialize Workspace Writes

- Wait for related read-only work before starting a writer.
- Never run `code_worker` and `code_expert` concurrently.
- Never run more than one workspace-writing agent.
- If a writer is active, wait for it to stop before starting another writer.
- Do not allow multiple agents to modify the same checkout concurrently.

## Boundaries

- Never change the primary model as part of routing.
- Never stage or commit changes unless the user explicitly requests it.
- Treat compilation and linking as build evidence, not runtime, correctness, or
  performance proof.
- Keep facts, inferences, and unverified risks distinct.
