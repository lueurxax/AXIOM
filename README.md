# AXIOM

AXIOM is a specification for an agent-native systems programming language designed for autonomous code generation, structural program transformation, and deterministic native compilation.

The repository is currently in the specification and planning stage. It does not yet contain a compiler or runtime implementation.

AXIOM is being designed around a canonical typed program graph, explicit effects, ownership-centered memory, structured concurrency, and machine-verifiable language semantics.

## Repository Contents

- `spec/agent-language-design.yaml` — foundational design principles and normative language requirements
- `spec/scope.yaml` — exact v1 scope boundaries, in-scope constructs, and explicit deferrals
- `axiom_language_creation_plan.md` — phased implementation roadmap
- `skills-required.md` — capability map for implementing AXIOM across phases
- `CLAUDE.md` — project-oriented repository notes for agent workflows

## Current State

- Repository maturity: specification and planning only
- Phase 0 semantics are being defined for modules, packages, and editions
- Phase 1 formal specification work has started
- The canonical typed program graph is the normative program representation
- Surface syntax is a derived artifact, not the source of truth

## Recommended Reading Order

1. `spec/scope.yaml`
2. `spec/agent-language-design.yaml`
3. `axiom_language_creation_plan.md`
4. `skills-required.md`

## Near-Term Deliverables

- Complete `spec/modules.yaml`
- Expand the spec set with `types.yaml`, `effects.yaml`, `errors.yaml`, `memory.yaml`, `concurrency.yaml`, `abi.yaml`, `determinism.yaml`, `validation.yaml`, and `ir.yaml`
- Begin Phase 2 work on the canonical program graph and patch model

## License

This repository is proprietary. See `LICENSE`.
