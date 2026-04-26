# IEF Architecture Context

> This document explains how this repository fits into the IEF (Intelligent Employee Framework) repository family.

## One-sentence role inside IEF

Binds IEF semantics into different agent ecosystems and tools -- the environment-binding layer for prompt injection, rule adaptation, and host-surface mapping.

## What this repository owns

- Prompt and rule injection for each host
- Host-surface mapping (agents.md, skills, hooks, system prompts)
- Environment conventions and native file mapping
- Runtime-specific bindings
- Host-specific packaging

## What this repository does NOT own

- Global task state (IEF-Work)
- Protocol schema design (IEF-Protocol)
- Core governance logic (IEF-Governance)
- Knowledge storage semantics (IEF-Knowledge)

## Dependencies on other IEF repos

| Dependency | Direction | Purpose |
|---|---|---|
| IEF-Governance | consumed | Governance packs to generate host-specific rules |
| IEF-Work | produced | Transforms external requests into internal work objects |
| IEF-Protocol | consumed | Shared schemas for task envelopes and handoffs |
| IEF-Knowledge | consumed | Memory and playbook references for host injection |
| IEF-Workers | produced | Worker capability declarations for adapter routing |

## Migration notes

This repository was formerly named `Harness-4-AIAgents`. It was renamed to `IEF-Adapters` as part of the IEF repository family reorganization (IEF Design Pack v0.2, section 19.3).

- The repository already functioned as the adapter/injection layer.
- Its current name was exploratory; the repository boundary is still correct.
- Rewritten README around host-environment adaptation (see root README.md).

## Current host surfaces

| Host | Status | Location |
|---|---|---|
| Hermes | Active | `adapters/hermes/` |
| OpenClaw | Active | `adapters/openclaw/` |
| Qoder | Active | `adapters/qoder/` |
| Claude Code | Planned | `adapters/claude_code/` |
| Codex | Planned | `adapters/codex/` |

## Related repositories

| Repository | Role |
|---|---|
| [IEF-Governance](https://github.com/brantzh6/IEF-Governance) | Governance layer |
| [IEF-Knowledge](https://github.com/brantzh6/IEF-Knowledge) | Memory and knowledge layer |
| [IEF-Work](https://github.com/brantzh6/IEF-Work) | Task and run state layer |
| [IEF-Protocol](https://github.com/brantzh6/IEF-Protocol) | Communication contracts |
| [IEF-Workers](https://github.com/brantzh6/IEF-Workers) | Concrete worker executors |
| [IEF-Adapters](https://github.com/brantzh6/IEF-Adapters) | This repo -- host-environment bindings |

## Full architecture reference

See [IEF Design Pack v0.2](https://github.com/brantzh6/IEF-Governance/blob/main/docs/IEF_Design_Pack_v0.2.md) for the complete architecture specification.
