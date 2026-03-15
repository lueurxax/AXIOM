# Required Skills for AXIOM Implementation

This document lists the skills required to implement the AXIOM language.
Each skill is mapped to specification requirements and to the phase of the plan where it is applied.

---

## Module System & Edition Model
*(Phase 0 - foundational, must complete before any implementation)*

| Skill | Description | Requirements |
|------|----------|-------|
| **Module system design** | Design `Module` as a first-class compilation unit: explicit exports/imports, visibility (`public` / `internal` / `private`), deterministic name resolution, no import side effects, no cyclic imports | MOD-001, MOD-002, MOD-005, MOD-SCOPE-001-006 |
| **Edition model** | Editions as named immutable snapshots of language semantics; gating for breaking changes; machine-detectable deprecation paths; agents must declare a target edition | MOD-004, P15 |
| **Machine-consumable spec authoring** | Author normative YAML specifications (`spec/modules.yaml` and the rest of `spec/*.yaml`): types, effects, errors, memory, concurrency, ABI, determinism, validation, IR | - |
| **Formal grammar design** | Define the language grammar (BNF/EBNF or PEG). **Optional in v1** - the parser is a derived artifact (REPR-003); grammar is needed only if a textual frontend is implemented | P9, REPR-003 |
| **Graph schema design** | Design JSON/YAML schemas for the program graph, patches, and diagnostics | ARCH-001 |

---

## Canonical Program Graph
*(Phase 2)*

| Skill | Description | Requirements |
|------|----------|-------|
| **Typed directed graph data structures** | Nodes, edges, stable node identity; edge kinds: `control_flow`, `data_flow`, `effect_flow`, `ownership_flow`, `task_dependency` (subkinds: `parent_child`, `cancellation`, `dependency`) | ARCH-001, ARCH-002 |
| **Graph canonicalization** | Algorithm for normalizing the graph into a unique form (`canonical form -> unique hash`) | ARCH-001 |
| **stable_id / semantic_id derivation** | **stable_id** - named nodes: `hash(node_type \| qualified_name_path \| type_signature)`; anonymous nodes: `hash(parent.stable_id \| edge_label \| edge_sequence_key \| node_type \| kind)`. `edge_sequence_key` is a monotonically increasing counter per `(parent, edge_label)` slot; existing nodes are NEVER reassigned on insert or reorder. Reordering updates `edge_order` only and MUST NOT change any `stable_id`. **semantic_id** - `hash(node_type \| kind \| type_signature \| canonical_semantic_payload)`; path-independent; used as a secondary key in `semantic_diff` to detect moved nodes. Gated on Spike 1. | ARCH-002, Spike 1 |
| **Graph serialization** | Binary and JSON serialization/deserialization of the CPG while preserving both IDs | ARCH-002 |

---

## Structural Patch Engine & Semantic Diff
*(Phase 3)*

| Skill | Description | Requirements |
|------|----------|-------|
| **Patch engine** | Implement the primitives `insert_node`, `replace_node`, `reconnect_edges`, `modify_contract`; define `stable_id` preservation rules for each operation | ARCH-003 |
| **Structural semantic diff** | Compute semantic differences between two CPGs; classify `added` / `removed` / `modified` / `moved` nodes by `stable_id` and `semantic_id` | ANA-002 |
| **Incremental validation** | Validate only the nodes and edges affected by a patch | ANA-001 |

---

## Type System & Ownership / Borrow Checker
*(Phase 4 - requires Spike 2 and Spike 3 before start)*

| Skill | Description | Requirements |
|------|----------|-------|
| **Type inference** | Deterministic type inference algorithm (Hindley-Milner or bidirectional) | TYPE-001 |
| **Algebraic data types & generics** | ADTs (sum/product types), parametric polymorphism | TYPE-001 |
| **Exhaustive pattern matching** | Match exhaustiveness checking; reject on `missing_variants` and `unreachable_branches` | TYPE-002 |
| **`Outcome<T,E>` type** | The sole fallible return type; prohibit `Result`, exceptions, null errors, and sentinel errors; `Outcome<T,Void>` is prohibited as an absence encoding | ERR-001, ERR-SCOPE-001-002 |
| **Ownership and move semantics** | Ownership modes: `owned` / `borrowed_shared` / `borrowed_exclusive` / `moved`; enforcement of MOVE-001 and MOVE-002 | MEM-SCOPE-003, MOVE-001, MOVE-002 |
| **Borrow checker** | Enforcement of BORROW-001, BORROW-002, ALIAS-001, ALIAS-002, and RACE-001; not optional in safe code | BORROW-001, BORROW-002, ALIAS-001, ALIAS-002, RACE-001, MEM-SCOPE-006 |
| **Unsafe block contracts** | Escape hatch with an explicit, auditable contract; borrow rules may be overridden only inside `unsafe` | P3, P4 |

---

## Region Memory Model
*(Phase 5 - layered on top of Phase 4 ownership discipline)*

| Skill | Description | Requirements |
|------|----------|-------|
| **Region/arena allocator design** | Static region model: explicit allocation scope and lifetime; each object belongs to exactly one region | MEM-001, MEM-002 |
| **Region lifetime analysis** | `region destroyed -> all objects invalid`; a region must not outlive its declared scope | MEM-001 |
| **Region+borrow integration** | Integrate the region validator with the borrow checker: `use-after-region`, cross-region exclusive aliasing, region escape | MEM-SCOPE-004, BORROW-002, Spike 3 |
| **Immutability enforcement** | Values are immutable by default; mutation requires explicit annotation | MEM-003, MEM-SCOPE-001, MEM-SCOPE-002 |

---

## Effect System & Capability Authority Model
*(Phase 6 - requires Spike 4 before full implementation)*

| Skill | Description | Requirements |
|------|----------|-------|
| **Effect inference & propagation** | Transitive propagation of effects across the call graph | EFF-001 |
| **Effect graph construction** | Build the effect dependency graph for capability enforcement | EFF-002 |
| **Capability model enforcement** | Access resources only through explicit capability objects (not ambient authority); CAP-001-CAP-005; authority boundaries: split / revoke / delegation; mockability for testing | EFF-003, CAP-001-005 |
| **Strict effect validation** | Reject unknown effects in strict mode; `reject_unknown_constructs: true` | EFF-001, MOD-004 |

---

## Structured Concurrency
*(Phase 7)*

| Skill | Description | Requirements |
|------|----------|-------|
| **Structured task model** | Parent-child task tree (CON-001: one parent per task); a parent MUST NOT complete while children are running (CON-002); propagating cancellation | CON-001, CON-002 |
| **Task dependency graph** | `parent_child` edges MUST form a tree; `cancellation` and `dependency` edges are separate structures; the validator enforces CON-001–CON-011 (CON-011 = task_dependency_graph scheduling rule) | CON-001-011 |
| **join / await / timeout semantics** | `join`, `await_all` (fail-fast: early exit on first failure, async cleanup; `collect_errors`: waits for all and returns `Collected(successes, errors)`), `await_any` (`all-failed` / `all-cancelled` / `empty`), `with_timeout` | CON-003-006 |
| **Region ownership transfer between tasks** | Transfer an owned `Region` through task inputs; static verification of CON-007-CON-010 | CON-007-010 |
| **Data race analysis** | Static proof of race freedom through task disjointness or synchronization primitives (RACE-001) | RACE-001 |

---

## SSA Intermediate Representation
*(Phase 8 - includes Spike 5 backend selection)*

| Skill | Description | Requirements |
|------|----------|-------|
| **SSA construction** | Direct lowering `CPG -> SSA`; phi-node placement, dominator tree. HIR is out of scope in v1; the normative pipeline is `CPG -> SSA -> native` | COMP-001 |
| **SSA IR design** | Design `axiom-ir`: closed instruction set, region/effect/capability annotations, ownership discipline preserved through lowering | COMP-001 |
| **IR optimizations** | Basic SSA optimizations: DCE, LICM, inlining, constant folding | PERF-001 |

---

## Native Code Generation & ABI
*(Phase 9 - backend selected by Spike 5)*

The backend is selected based on Spike 5: both are prototyped, one is chosen, and the other is deferred.

| Skill | Description | Requirements |
|------|----------|-------|
| **LLVM integration** | Generate LLVM IR, target triple, ABI mapping, and optimization passes. Required for the Spike 5 prototype. | COMP-002 |
| **Cranelift integration** | `cranelift-codegen`: fast compile time. Required for the Spike 5 prototype. | COMP-002 |
| **Backend evaluation (Spike 5)** | Compare LLVM and Cranelift by compile time; choose one; document the kill criterion | Spike 5 |
| **C ABI interop** | Layout, calling convention, and ownership transfer across the FFI boundary | ABI-001 |
| **Deterministic code generation** | Reproducible binary outputs; layout determinism | DET-001, PERF-002 |

---

## Minimal Runtime
*(Phase 10)*

| Skill | Description | Requirements |
|------|----------|-------|
| **Runtime region allocator** | Run-time arena allocator without GC | PERF-003 |
| **Minimal task scheduler** | Cooperative / structured async runtime without green threads | CON-001, PERF-003 |
| **Root capability constructors** | Exactly one constructor for each non-structural effect (CAP-006); only in the root entrypoint (CAP-003): `acquire_fs_read_cap`, `acquire_fs_write_cap`, `acquire_network_read_cap`, `acquire_network_write_cap`, `acquire_database_read_cap`, `acquire_database_write_cap`, `acquire_time_cap`, `acquire_random_cap`, `acquire_process_cap`, `acquire_unsafe_ffi_cap`. `region_alloc` and `pure` have no constructors. | EFF-003, CAP-003, CAP-006 |
| **System interface / syscall layer** | Minimal syscall bindings; prohibit a large libc runtime; prohibit ambient authority | PERF-003 |

---

## Compiler & Agent API
*(Phase 11a - graph-level, runs in parallel to Phases 8-10; Phase 11b - compilation API, requires backend)*

| Skill | Description | Requirements |
|------|----------|-------|
| **Compiler API design** | Library / RPC API: `parse_graph`, `validate_graph`, `apply_patch`, `semantic_diff`, `incremental_verify`, `typecheck`, `lower_to_ssa`, `compile` | P11 |
| **Structured JSON diagnostics** | Machine-readable diagnostics: `diagnostic_id`, `severity`, `requirement` (e.g. `BORROW-001`), `node_stable_id`, `suggested_fix`; not free-form text | P11, tooling_contract |
| **Incremental compilation API** | Incremental queries; validation cache | ANA-001 |
| **Query and summary APIs** | `query_symbols`, `query_dependencies`, `query_effect_summary`, `query_borrow_summary`, `query_capability_requirements` - required for the Phase 12 agent workflow | P11, ANA-001, ANA-002 |
| **Text pipeline (optional)** | `print(graph) -> text` (printer, REPR-002); `format(text) -> canonical_text` (formatter, REPR-004); `parse(canonical_text) -> graph` (parser, REPR-003). `format(graph)` is an invalid call. Round-trip `parse(format(print(g))) == g` only when both printer and parser exist. | REPR-003, REPR-004, REPR-005 |
| **Optimization remarks API** | `optimization_remarks(graph, level) -> RemarkSet`; keyed by `stable_id`; machine-readable | PERF-001 |

---

## Agent Synthesis SDK
*(Phase 12)*

| Skill | Description | Requirements |
|------|----------|-------|
| **Agent synthesis workflow** | `Generate graph -> patch -> validate -> compile -> deploy`; the agent declares an edition, validates in strict mode, and fixes errors from structured diagnostics | MOD-004, ANA-001, ANA-002, COMP-002 |
| **Edition targeting** | Stable targeting of a specific edition by agents (`axiom-v1`); `reject_unknown_constructs` | MOD-004, P15 |
| **Strict mode enforcement** | `reject_unknown_constructs: true`; audit capability boundaries; all features MUST satisfy specification requirements | EFF-003, MOD-004, P3 |

---

## Research Spikes

Spikes are kill-criterion-gated research tracks. A phase cannot begin until its leading spikes have passed.

| Spike | Gates | Skill | Kill criterion |
|-------|-----------|------|----------------|
| **Spike 1 - stable_id collision resistance** | Canonical Program Graph | Cryptographic hashing, empirical collision analysis | Collision rate > 1 in 10^12 -> revise the hash scheme |
| **Spike 2 - ownership/borrow inference tractability** | Type System & Ownership | Type systems research, ownership inference prototyping | Exponential time on realistic programs -> revise the type system design |
| **Spike 3 - region+borrow interaction correctness** | Type System & Region Model | Formal methods, proof checking (Coq/Lean/TLA+), core calculus | Soundness counterexamples -> fix the interaction rules before implementation |
| **Spike 4 - capability delegation performance** | Effect System & Concurrency | Performance engineering, task graph micro-benchmarking | Overhead > 5% of task scheduling cost -> optimize the representation |
| **Spike 5 - backend selection** | SSA IR & Native Codegen | LLVM + Cranelift prototyping, compile-time benchmarking | Prototype both, choose one, defer the other |

---

## Cross-Cutting Skills

| Skill | Application |
|------|------------|
| **Property-based testing** | Verify invariants of the CPG, type system, effects, and patches |
| **Fuzzing** | Search for memory safety violations and invariant breaks |
| **Benchmarking** | Zero-cost abstractions, layout determinism (PERF-001) |
| **Reproducible builds** | DET-001 - deterministic compilation in CI/CD |
| **Spec conformance testing** | Automated verification that the implementation conforms to the YAML specification |
| **Structured logging / observability** | Telemetry, tracing, structured panic reports |
| **Formal methods** | Spike 3 (required); recommended for Type System & Region Model soundness |

---

## Compiler Implementation Language

| Option | Pros | Cons |
|---------|-------|--------|
| **Rust** | Ownership/lifetimes align with AXIOM semantics; `inkwell` (LLVM), `cranelift-codegen` | Higher entry barrier |
| **Go** | Simplicity, good graph tooling support, fast iteration | No ownership model; GC |
| **C++** | Direct integration with the LLVM API | Unsafe, complex |

Rust is the preferred option: the implementation language semantics are the closest to AXIOM semantics.
