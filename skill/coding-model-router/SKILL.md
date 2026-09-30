---
name: coding-model-router
description: Select roles for bounded, separable coding subtasks when delegation has a concrete benefit or the user explicitly requests routing.
---

# Coding Model Router

Keep the primary agent responsible for the requested outcome, including difficult analysis, implementation, necessary checks, and corrections. Invoking this skill does not require spawning an agent.

## Decide Whether to Delegate

Use delegation for a concrete subtask that can be separated from the primary agent's useful work: independent evidence gathering, a sufficiently large fully specified mechanical batch, or an independent review.

Keep focused searches, configuration changes, local fixes, routine tests and builds, and closely coupled algorithm or debugging work in the primary agent. Complexity, testing, and file count alone are not delegation triggers. If a handoff adds more coordination than value, continue directly.

## Choose a Role

| Bounded subtask | Role |
|---|---|
| Independent read-only definitions, references, call chains, or log evidence | `code_reader` |
| Fully specified mechanical changes or routine execution whose size or duration makes delegation useful | `code_worker` |
| Independent difficult analysis, an explicitly assigned implementation, or requested expert review | `code_expert` |

The expert role is an optional peer, not a mandatory escalation destination. Readers and workers return evidence and out-of-scope decisions to the primary agent.

## Dispatch Boundaries

- State the expected benefit and give the selected agent its exact goal, scope, evidence, ownership, and validation requirements.
- Honor explicit user-selected agents and read-only constraints. Check currently available role/model definitions before dispatch; do not change the primary model merely to route work.
- Use the current tool schema for fork, model, and reasoning parameters. Old examples do not establish that full-history forks accept overrides.
- Parallelize only mutually independent read-only tasks. Before writing, wait for readers whose evidence depends on the files being changed.
- Allow one workspace writer at a time, including the primary agent. Never run `code_worker` and `code_expert` concurrently. Tell writers that they are not alone and must preserve others' edits.
- Subagents do not delegate further or broaden the task. Review their evidence before relying on material conclusions; repeat verification only for an unresolved concern.
- Do not stage or commit changes unless explicitly requested. Keep static, build, runtime, visual, and performance evidence distinct.
