# AXIOM

## Project Overview

AXIOM is an **agent-native, high-performance systems programming language** designed to be generated, transformed, and executed by autonomous agents rather than humans.

The repository is currently in the **specification and planning phase**. No implementation code exists yet.

## Repository Structure

```
AXIOM/
├── README.md                        # Primary repository entry point
├── CLAUDE.md                        # Project notes for agent workflows
├── docs/
│   ├── README.md                    # Human-facing documentation index
│   ├── architecture/
│   │   └── architecture-comparison.md
│   └── planning/
│       ├── language-creation-plan.md
│       └── required-skills.md
└── spec/
    ├── agent-language-design.yaml   # Foundational language design specification
    └── scope.yaml                   # Exact AXIOM v1 scope and exclusions
```

## Core Language Properties

- Canonical typed program graph (CPG) as primary program representation
- Patch-first architecture (`insert_node`, `replace_node`, `reconnect_edges`, `modify_contract`)
- Region/arena memory model — no GC, no implicit heap allocation
- Explicit effect system — all functions declare effects in their signature
- Unified `Outcome<T,E>` error model — no exceptions, no null errors
- Strong static typing with exhaustive pattern matching
- Structured concurrency via parent-child task graphs
- Zero-cost abstractions
- Deterministic, reproducible compilation
- Direct C ABI interoperability
- Native code generation (LLVM or Cranelift backend)

## Design Principles (Summary)

Key normative rules from `spec/agent-language-design.yaml`:

| ID    | Title                        | Level  |
|-------|------------------------------|--------|
| P1    | Deterministic by default     | MUST   |
| P2    | Explicit effects             | MUST   |
| P3    | Safety by default            | MUST   |
| P4    | No UB in safe code           | MUST   |
| P5    | Ownership-centered resources | SHOULD |
| P6    | Structured concurrency       | MUST   |
| P7    | Errors are values            | MUST   |
| P8    | Capability-based side effects| MUST   |
| P10   | Zero-cost abstractions       | MUST   |

## Specification Requirement IDs

Requirements use stable identifiers that agents reference during synthesis and validation:

- `ARCH-*` — Core architecture (program graph, node identity, patch model)
- `TYPE-*` — Type system
- `MEM-*`  — Memory model
- `EFF-*`  — Effect system
- `ERR-*`  — Error model
- `CON-*`  — Concurrency
- `PERF-*` — Performance
- `ANA-*`  — Program analysis
- `COMP-*` — Compilation
- `ABI-*`  — Interoperability
- `DET-*`  — Determinism

## Implementation Phases

| Phase | Topic                       | Key Deliverable                 |
|-------|-----------------------------|---------------------------------|
| 0     | Modules, Packages, Editions | `spec/modules.yaml`             |
| 1     | Core Specification          | Full `spec/` YAML suite         |
| 2     | Canonical Program Graph     | CPG schema + library            |
| 3     | Patch System                | `apply_patch`, `semantic_diff`  |
| 4     | Type System                 | `axiom-typecheck`               |
| 5     | Region Memory System        | Region validator                |
| 6     | Effect System               | Effect analyzer                 |
| 7     | Concurrency Model           | Task dependency graph validator |
| 8     | Intermediate Representation | `axiom-ir`, `axiom-lowering`    |
| 9     | Backend                     | `axiom-compiler`, `axiom-abi`   |
| 10    | Minimal Runtime             | `axiom-runtime`                 |
| 11    | Agent Tooling               | Compiler API for agents         |
| 12    | Agent SDK                   | Full agent synthesis workflow   |

**Critical path:** Phase 0 (module semantics) + Phase 2 (CPG) + Phase 3 (patch engine) + validation are foundational for all later phases.

## First Milestone Target

An agent-operable system that can:

1. Create a program graph
2. Apply structural patches
3. Run incremental validation
4. Compute semantic diff
5. Compile to a native binary

Initial language subset: functions, records, pattern matching, regions, `Outcome<T,E>`, explicit effects.

## Planned Spec Files (Phase 1 Deliverables)

```
spec/
  agent-language-design.yaml  ✓ (exists)
  scope.yaml                  ✓ (exists)
  modules.yaml
  types.yaml
  effects.yaml
  errors.yaml
  memory.yaml
  concurrency.yaml
  abi.yaml
  determinism.yaml
  validation.yaml
  ir.yaml
```

## Agent Execution Rules

From `spec/agent-language-design.yaml`:

- All generated features MUST satisfy spec requirements
- Violations MUST be rejected during validation
- Strict mode: `reject_unknown_constructs: true`
- Generated code targets a declared language edition (P15)
