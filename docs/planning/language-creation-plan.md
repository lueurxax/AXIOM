
# AXIOM Language Creation Plan

## Overview

AXIOM is an **agent-native, high-performance systems programming language** designed to be generated, transformed, and executed by autonomous agents rather than humans.

Core properties:

- Canonical typed program graph representation
- Patch-first architecture
- Region/arena memory model with ownership and borrow discipline
- Explicit effect system with capability-based authority
- Unified `Outcome<T,E>` error model (no Result, no Option as separate error channel)
- Strong static typing with ownership/move/alias discipline
- Structured concurrency with operational join/await/timeout/failure-propagation semantics
- Zero-cost abstractions
- Deterministic compilation
- Explicit module/package/edition semantics
- Direct C ABI interoperability
- Native code generation

Primary goals:

- reliable synthesis by AI agents
- structural program transformation
- incremental verification and semantic diff
- machine-verifiable correctness
- predictable high performance execution

---

# Phase 0 — Module, Package, and Edition Semantics

## Goal

Define **module system, visibility rules, dependency resolution, and language evolution model** before any implementation begins. These semantics are foundational for reproducible compilation and agent-targeted code generation.

## Motivation

Without explicit module/package/edition semantics, reproducible builds are impossible, import-order side effects can silently change program meaning, and code-generation agents cannot reliably target a stable language version.

## Deliverables

### Module system specification

```
spec/modules.yaml
```

Core concepts:

```
Module          — named unit of declarations with explicit visibility
Package         — versioned collection of modules with a manifest
Edition         — named language version gate for breaking changes
ImportGraph     — directed acyclic graph of package dependencies
```

### Visibility rules

```
public          — visible outside the module
internal        — visible within the package
private         — visible within the module only
```

The compiler MUST reject references that violate visibility boundaries.

### Dependency resolution (v1 scope: local compilation units only)

v1 does NOT include a package manager, remote dependencies, or a version solver.
These are explicitly out of scope (MOD-SCOPE-007, MOD-SCOPE-008, MOD-SCOPE-009).

v1 module system covers:

```
explicit imports and exports within local compilation units
deterministic name resolution
no import side effects
no cyclic imports
```

Future editions will add:

```
manifest.axiom  — machine-readable package manifest
lock.axiom      — deterministic lockfile (content-addressed)
remote dependency resolution
vendoring for offline/reproducible builds
```

### Edition model

```
edition: "axiom-v1"
```

Rules:

```
each edition is a named, immutable snapshot of language semantics
breaking changes MUST be gated by a new edition identifier
code generation agents MUST declare a target edition
deprecation paths MUST be machine-detectable
```

### Requirement IDs introduced

```
MOD-001  modules are explicit named units (v1: MUST)
MOD-002  visibility is declared, not ambient (v1: MUST)
MOD-003  dependency resolution is deterministic and lockable (deferred: out of v1 scope)
MOD-004  editions gate breaking changes (v1: MUST)
MOD-005  the build graph is acyclic and order-independent (v1: MUST)
```

---

# Phase 1 — Formalize the Core Specification

## Goal

Produce a **machine-consumable language specification** defining semantics and constraints.

## Deliverables

### Specification repository

```
spec/
  agent-language-design.yaml
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

### Formal grammar schema

Core constructs:

```
Program
Module
Function
Block
Expression
Pattern
Type
Effect
Region
Task
```

### Program graph schema

Programs are defined as a **typed directed graph**. The node set is unified across all phases — Phase 1 specification and Phase 2 implementation MUST use the same canonical node taxonomy.

```yaml
node_types:
  Module
  Function
  Block
  Expression
  Pattern
  Type
  Effect
  Region
  Task

edges:
  control_flow
  data_flow
  effect_flow
  ownership_flow
  task_dependency

identity:
  stable_node_identity: true
  stable_id_derivation: semantic_identity_with_persistent_lineage
```

**Note on Binding:** `Binding` is not a separate node type. A binding is represented as an `Expression` node of kind `let_binding`, carrying `name`, `type`, `region`, and `ownership_mode` properties. This unification ensures that stable_id computation and semantic_diff operate on a consistent graph.

### Node identity: stable_id and semantic_id

Each node carries two complementary identities:

**stable_id** — encodes WHICH semantic node this is across edits:

```
named nodes (Module, Function, Type, Effect):
  stable_id = hash(node_type | declaring_scope_lineage | semantic_name_seed)

anonymous nodes (Block, Expression, Pattern, Task):
  stable_id = hash(node_type | lineage_anchor | identity_seed)
```

`identity_seed` is allocated exactly once when a semantic node first appears in the
canonical graph. It is persisted by the patch engine and survives:

```
- sibling insertion
- sibling deletion
- sibling reorder
- pretty-print / re-layout
- reserialization
- canonical field reorder
```

`stable_id` MUST change only when the old semantic node ceases to exist:

```
- node replaced by a different semantic entity
- node split into multiple entities
- node merged into another entity
- normalization explicitly destroys the old semantic node
```

Canonical ordering and edge ordering are tracked separately and MUST NOT be used as
identity.

**semantic_id** — encodes WHAT the node currently means:

```
semantic_id = hash(node_type | kind | type_signature | canonical_semantic_payload)
```

`edge_label` and `kind` MUST be drawn from a closed enumeration in the node taxonomy. Unknown values MUST be rejected by the validator.

Rules:

```
1. stable_id MUST NOT depend on node_id (runtime-assigned, positional)
2. stable_id MUST NOT depend on sibling_index, edge_order, or any position-derived counter
3. replace_node: preserve stable_id iff the replacement is explicitly declared
   to be the same semantic node; otherwise allocate a new stable_id and tombstone the old one
4. insert_node: allocate a fresh identity_seed for the new node;
   NO OTHER existing node's stable_id changes
5. reconnect_edges: MUST NOT change stable_id or semantic_id of any unaffected node
6. reorder_siblings: MUST NOT change stable_id of any node; updates edge_order only
7. semantic_diff uses stable_id as primary key, semantic_id as secondary key
8. incremental_verify treats matching stable_id as identity-equal across patches
```

The hash function MUST be deterministic, collision-resistant, and version-pinned per edition. See Research Spike 1.

### Requirement IDs

All rules must have stable identifiers.

Examples:

```
ARCH-001
TYPE-001
MEM-001
EFF-001
ERR-001
CON-001
DET-001
MOD-001
```

Agents reference these during synthesis and validation.

---

# Phase 2 — Canonical Program Graph (CPG)

## Goal

Define the **primary representation of AXIOM programs**.

Programs are stored as a typed program graph. The schema defined here is the canonical implementation of the node taxonomy from Phase 1 — it MUST be identical.

### Node types

```
Module
Function
Block
Expression    (includes let_binding as a kind, not a separate node type)
Pattern
Type
Effect
Region
Task
```

### Edge types

```
control_flow
data_flow
effect_flow
ownership_flow
task_dependency
```

### Node properties

```
node_id         runtime-assigned, positional, not stable
stable_id       persistent semantic node identity per Phase 1 derivation algorithm
                encodes WHICH semantic node this is across edits; survives
                sibling insertion/deletion/reorder, reserialization
semantic_id     content hash of the node's current semantic payload
                encodes WHAT the node currently means; changes whenever
                type_signature, body, effects, or capabilities change
                required for moved_node detection in semantic_diff (ARCH-003)
type
effect
region
capabilities    list of capability object references (see Phase 6)
                MUST be a list; never a scalar. Empty list means no
                ambient authority (pure or region_alloc only).
contract
ownership_mode  owned | borrowed_shared | borrowed_exclusive | moved
```

### Example node

```json
{
  "node_id": "fn_42",
  "stable_id": "sha256:e3b0c44298fc1c149afb...",
  "semantic_id": "sha256:a4c2f1e9b3d70812c5...",
  "node_type": "Function",
  "inputs": [],
  "outputs": [],
  "effects": ["filesystem_read"],
  "region": "r1",
  "capabilities": ["cap:fs.read.0x1a2b"],
  "ownership_mode": "owned"
}
```

---

# Phase 3 — Patch System

## Goal

Allow agents to modify programs via **structural patches**.

### Patch primitives

```
insert_node
replace_node
reconnect_edges
modify_contract
```

Derived patch helpers may be added later, but they must lower to the canonical primitives above.

### stable_id preservation rules under patch operations

```
replace_node(target, new_node):
  - if new_node is explicitly declared the same semantic entity as target:
      new_node.stable_id = target.stable_id   (identity preserved)
  - otherwise:
      new_node.stable_id = allocate_new_semantic_identity()   (new identity)
  - the old stable_id becomes tombstoned in the patch log

insert_node(parent, position, new_node):
  - new_node.stable_id = allocate_new_semantic_identity()
  - no other existing node's stable_id changes
  - edge_order is updated to include new_node at the declared position

reconnect_edges(node, old_edge, new_edge):
  - does not change stable_id of any unaffected node
  - records edge provenance in patch log

modify_contract(node, contract_delta):
  - contract changes recompute semantic_id
  - stable_id changes only if the old semantic node is explicitly replaced
```

### Patch example

```json
{
  "operation": "replace_node",
  "target_stable_id": "sha256:e3b0c44298fc1c149afb...",
  "new_node": {},
  "stable_id_policy": "preserve_if_same_semantic_entity"
}
```

### Deliverables

Patch engine functions:

```
apply_patch(program, patch)
incremental_validate(program, patch)
semantic_diff(before, after)
```

`semantic_diff` uses `stable_id` as primary key and `semantic_id` as secondary key. It MUST identify:

```
added_nodes     — stable_id present in after, absent in before

removed_nodes   — stable_id present in before, absent in after,
                  AND no node in after has the same semantic_id

modified_nodes  — same stable_id in before and after,
                  one or more properties changed

moved_nodes     — stable_id absent in after, but a node in after
                  has the same semantic_id at a different structural position;
                  if semantic_id is ambiguous (multiple matches), report as
                  removed_nodes + added_nodes instead
```

---

# Phase 4 — Type System

## Goal

Implement the **static type checker** including ownership and borrow discipline.

### Supported types

```
primitive types
records
algebraic data types
generics
Outcome<T,E>      (unified fallible result; the ONLY fallible return type)
```

**Removed:** `Option` and `Result` are NOT separate types in v1. The unified error model (ERR-001) encodes all fallible and optional operations as `Outcome<T, E>`.

Absence is modeled by choosing an error type that names the absence condition:

```
Outcome<T, NotFound>     — value may not exist (e.g. map lookup)
Outcome<T, Empty>        — collection may be empty
Outcome<T, Unset>        — optional field may be unset
```

`Outcome<T, Void>` MUST NOT be used to model absence. `Void` is uninhabited; an `Outcome<T, Void>` collapses to `Success(T)` and is therefore infallible, not optional. Using it as an absence encoding is a type error in intent.

If a function is truly infallible, its return type is `T`, not `Outcome<T, Void>`.

### Ownership and borrow discipline

Regions alone do not guarantee memory safety or data-race freedom. The type system MUST enforce:

```
ownership modes:
  owned               — value has unique owner; moving transfers ownership
  borrowed_shared     — read-only reference; multiple may coexist
  borrowed_exclusive  — read-write reference; no other borrows active
  moved               — value has been consumed; subsequent use is a type error

rules:
  MOVE-001  moving an owned value invalidates the source binding
  MOVE-002  a function that takes ownership MUST be called with an owned value
  BORROW-001 borrowed_shared and borrowed_exclusive CANNOT coexist on the same value
  BORROW-002 a borrowed reference MUST NOT outlive its source region
  ALIAS-001  two borrowed_exclusive aliases to overlapping memory are a type error
  ALIAS-002  aliasing is safe only for borrowed_shared or for disjoint regions
  RACE-001   cross-task access to a value requires either:
               (a) borrowed_shared with proof of task disjointness, or
               (b) explicit synchronization primitive
```

These rules are enforced at compile time. Unsafe blocks MAY override them with explicit contracts.

### Requirements

- static typing
- deterministic type inference
- incremental validation
- exhaustive pattern matching
- no dynamic typing
- ownership/borrow checking is not optional in safe code

### Deliverable

```
axiom-typecheck  (includes ownership/borrow checker)
```

---

# Phase 5 — Region Memory System

## Goal

Implement **region/arena memory management** in coordination with the ownership discipline from Phase 4.

### Core elements

```
Region
Allocation
Region lifetime
Region ownership
```

### Rules

```
object belongs to exactly one region
region destroyed → all objects in region are invalid
allocation is explicit: stack | region | explicit_heap
values are immutable by default
region ownership can be transferred between tasks (see Phase 7)
```

### Compiler checks

These checks operate on top of the ownership rules from Phase 4:

```
use-after-region          — reference to value whose region has been destroyed
cross-region alias         — borrowed_exclusive alias spanning two regions
invalid region lifetime   — region outlives its declared scope
region-escape             — reference escapes the region that owns it
```

### Relationship to memory safety

Regions define the **allocation and deallocation discipline**. Ownership/borrow rules (Phase 4) define the **alias and lifetime discipline**. Both are required for memory safety. Neither is sufficient alone.

### Deliverable

```
region validator (integrated with Phase 4 ownership checker)
```

---

# Phase 6 — Effect System and Capability Authority Model

## Goal

Track all side effects statically **and enforce capability-based authority**.

### Effect types

```
pure
region_alloc
filesystem_read
filesystem_write
network_read
network_write
database_read
database_write
process_spawn
time
randomness
unsafe_ffi
```

### Capability authority model

Effects alone do not prevent ambient authority. The capability model makes authority **explicit, passable, and auditable**.

#### Capability objects

A capability object is a first-class value that confers authority to perform a specific effect. It is NOT ambient — it MUST be explicitly passed as a function argument or field.

Every effect in the effect inventory MUST have a corresponding capability object.
The mapping is one-to-one: no effect may be declared without a backing capability type.

```
Effect              Capability object   Notes
──────────────────  ──────────────────  ──────────────────────────────────────────
filesystem_read     FsReadCap           read from filesystem
filesystem_write    FsWriteCap          write to filesystem
network_read        NetworkReadCap      read from network sockets / HTTP
network_write       NetworkWriteCap     write to network sockets / HTTP
database_read       DatabaseReadCap     read from a database connection
database_write      DatabaseWriteCap    write to a database connection
time                TimeCap             read the clock
randomness          RandomCap           read entropy source
process_spawn       ProcessCap          spawn child processes
unsafe_ffi          UnsafeFfiCap        call unsafe foreign functions
region_alloc        (implicit)          governed by MEM-* rules; no external cap needed
pure                (none)              no authority required
```

`region_alloc` is governed by the region/ownership system (MEM-001 through MEM-004) rather
than a capability object; it is structurally checked, not authority-checked.
`pure` requires no capability.

All other effects require an explicit capability object held by the function.

Rules:

```
CAP-001  a function that declares effect E MUST receive a capability object for E
CAP-002  capability objects are owned values; passing one transfers authority
CAP-003  capability objects cannot be forged; they are created only at system boundaries
CAP-004  a function MUST NOT perform effect E without holding a matching capability
CAP-005  capability objects are mockable for testing
```

#### Authority boundaries

```
system_root_capabilities  — created at program entry point (main or agent entrypoint)
capability_split          — capability MAY be narrowed (e.g., read-only sub-cap from read-write cap)
capability_revoke         — capability MAY be dropped; dropped cap cannot be used
capability_delegation     — capability MAY be passed to child tasks (see Phase 7)
```

#### System interface contract

The root constructors are the ONLY source of capability objects. The mapping between
constructors and effect/capability types MUST be exhaustive: every non-structural
effect in the inventory MUST have a corresponding constructor.

```
acquire_fs_read_cap(path_prefix)              -> FsReadCap
acquire_fs_write_cap(path_prefix)             -> FsWriteCap
acquire_network_read_cap(host_allow_list)     -> NetworkReadCap
acquire_network_write_cap(host_allow_list)    -> NetworkWriteCap
acquire_database_read_cap(connection_spec)    -> DatabaseReadCap
acquire_database_write_cap(connection_spec)   -> DatabaseWriteCap
acquire_time_cap()                            -> TimeCap
acquire_random_cap()                          -> RandomCap
acquire_process_cap(allowed_commands)         -> ProcessCap
acquire_unsafe_ffi_cap(allowed_symbols)       -> UnsafeFfiCap

No library code MAY call these constructors; only the root entrypoint MAY do so.
```

CAP-006 (completeness invariant): the set of capability types MUST be exactly the
set of non-structural effects. A validator MUST reject any effect declaration for
which no capability type exists, and MUST reject any capability type for which no
effect exists.

### Requirements

- effects declared in function signature
- effects propagate through call graph
- compiler enforces effect boundaries
- effect graph drives capability enforcement
- unknown effects rejected in strict validation mode

### Deliverable

```
effect analyzer (includes capability flow checker)
```

---

# Phase 7 — Concurrency Model

## Goal

Implement **structured concurrency** with full operational semantics.

### Task node structure

```
Task
parent
inputs
outputs
region
effects
capabilities        list of capability objects delegated to this task
cancellation_scope
```

### Dependency edges

```
parent_child
cancellation
dependency
```

### Operational semantics

The structured concurrency model MUST define the following operationally, not just declaratively:

#### Join and await semantics

```
spawn(task) -> TaskHandle<T,E>

join(handle) -> Outcome<T, TaskError<E>>:
  blocks the calling task until the child task completes
  returns the child's return value or propagates its error

await_all(handles) -> Outcome<Vec<T>, TaskError<E>>:
  default (fail-fast): blocks until either (a) all children succeed, or (b) any child fails.
  On (a): returns Success(Vec<T>) in spawn order.
  On (b): sends cancellation to all remaining running children, then blocks until
    all cancelled children have completed (including cleanup), THEN returns
    Failure(TaskError<E>) with the first failure observed.
    The parent does NOT resume until every child — including cancelled ones — has
    fully terminated. This preserves CON-002: a parent MUST NOT complete while
    any child task is still running.

await_all(handles, collect_errors: true)
    -> Outcome<Vec<T>, TaskError::Collected(successes: Vec<T>, errors: Vec<TaskError<E>>)>:
  opt-in collection mode: blocks until ALL child tasks complete; does NOT cancel on failure.
  returns Success(Vec<T>) if all succeed (values in spawn order);
  returns Failure(TaskError::Collected(successes, errors)) on any failure,
    where successes holds return values of successful children (failed positions absent)
    and errors holds one TaskError<E> per failed child in completion order.
  Both fields MUST be populated by the runtime.

await_any(handles) -> Outcome<T, TaskError<E>>:
  blocks until the first child task completes successfully;
  sends cancellation to remaining children once a success is observed,
  then blocks until all cancelled children have fully terminated (including
  cleanup), THEN returns Success(T). The parent does NOT resume until every
  child has fully terminated. Consistent with CON-002.

  If all children fail before any succeeds:
    returns Failure(TaskError::AllFailed(Vec<TaskError<E>>))
    containing the errors of all failed children in completion order.

  If all children are cancelled (e.g. via external cancellation scope)
  before any succeeds:
    returns Failure(TaskError::Cancelled)

  If handles is empty:
    returns Failure(TaskError::AllFailed([]))  -- no children to succeed
```

#### Timeout model

```
with_timeout(duration, task) -> Outcome<T, TimeoutError>:
  cancels the task if it does not complete within duration
  cancellation propagates to all children of the task
  timeout is expressed in logical time units to preserve determinism
  wall-clock time MAY differ; requires TimeCap
```

#### Structured task invariants

```
CON-001  every task MUST have exactly one parent task, except the root task
         which has no parent; the resulting task graph MUST be a tree
CON-002  a parent task MUST NOT complete (succeed or fail) while any child
         task is still running; it MUST either join all children or cancel them
         and then wait for all cancelled children to fully terminate (including
         cleanup) before the parent itself returns. There is no distinction
         between "semantic completion" and "cleanup completion" for this rule:
         a child is considered done only when it has fully terminated.
```

#### Failure propagation

```
CON-003  if a child task fails and the parent does not handle the error,
         the parent MUST be cancelled (fail-fast default)
CON-004  the parent MAY opt into error collection mode via await_all with
         collect_errors = true
CON-005  a panic in a child task is always propagated to the parent as TaskError::Panic
CON-006  orphaned tasks (detached execution) MUST be lexically marked with
         detach! and are NOT covered by structured guarantees
```

#### Region ownership transfer between tasks

```
CON-007  a parent task MAY transfer ownership of a region to a child task
         by passing an owned Region value as a task input
CON-008  once a region is transferred to a child, the parent MUST NOT
         access any value borrowed from that region
CON-009  when the child task completes, the region is either:
         (a) returned to the parent as a task output (ownership transferred back), or
         (b) destroyed by the child (if the child owns it at completion)
CON-010  borrowed_shared references to a region MAY be shared across tasks
         only if the region outlives all tasks that hold references to it
```

These rules make `parent_scope_owns_children` statically verifiable rather than merely asserted.

### Deliverable

```
task scheduler specification (operational semantics document)
task dependency graph validator (enforces CON-001 through CON-011)
```

---

# Phase 8 — Intermediate Representation

## Goal

Lower the canonical program graph into **SSA-based IR** suitable for backend code generation.

### Compilation pipeline

```
CPG  (canonical typed program graph — sole normative representation)
↓  axiom-lowering  (CPG → SSA; Phase 8 deliverable)
SSA  (static single-assignment IR; schema and invariants defined here)
↓  backend (LLVM / Cranelift; Phase 9 deliverable)
native binary
```

HIR is **not in scope for v1**. A named HIR stage with a formal schema,
invariants, and acceptance criteria may be introduced in a future edition
if the CPG → SSA gap proves too large. Until then, adding an undefined HIR
to the pipeline creates an unverifiable semantic drift point and is
explicitly deferred.

### SSA IR specification (Phase 8 deliverable)

The SSA IR MUST define:

```
schema:
  - instruction set (closed enumeration; no undefined opcodes)
  - basic block structure and dominator-tree invariants
  - phi-node placement rules
  - region lifetime annotations carried through lowering
  - effect annotations carried through lowering (from EFF-001)
  - capability annotations carried through lowering (from EFF-003)

invariants enforced after CPG → SSA lowering:
  - every def dominates all uses
  - effect set of a lowered function equals effect set declared in CPG
  - region lifetimes are preserved (no use-after-region, no cross-region alias)
  - ownership discipline is preserved (no double-free, no use-after-move)

acceptance criteria (Phase 8 exit):
  - SSA IR validates against the schema for all Phase 4–7 test programs
  - region and effect annotations survive lowering unchanged
  - Spike 5 backend comparison uses this IR as the stable interface
```

### Deliverables

```
axiom-ir        (SSA IR schema + validator)
axiom-lowering  (CPG → SSA lowering pass)
```

---

# Phase 9 — Backend

## Goal

Generate optimized native code with deterministic layouts and direct C ABI interoperability.

### Backend options

LLVM:
- mature optimizations
- cross-platform

Cranelift:
- fast compile times
- simpler integration

### Deliverable

```
axiom-compiler
axiom-abi
```

---

# Phase 10 — Minimal Runtime

## Goal

Provide minimal runtime support.

### Runtime components

```
region allocator
task scheduler
system interface (provides root capability constructors)
```

### Prohibited runtime features

```
garbage collection
dynamic reflection
large runtime frameworks
ambient authority to filesystem/network/clock/entropy
```

### Deliverable

```
axiom-runtime
```

---

# Phase 11 — Agent Tooling

## Goal

Expose compiler functionality to agents via APIs with **structured, machine-consumable outputs**.

### Required interfaces

```
parse_graph
validate_graph
apply_patch
semantic_diff
incremental_verify
typecheck
lower_to_ssa
compile
```

### Structured diagnostics

All errors and warnings MUST be emitted as structured JSON diagnostics, not free-form text:

```json
{
  "diagnostic_id": "TYPE-042",
  "severity": "error",
  "requirement": "BORROW-001",
  "node_stable_id": "sha256:e3b0c...",
  "message": "borrowed_exclusive and borrowed_shared coexist on value 'x'",
  "span": { "module": "core::parser", "line": 42, "col": 8 },
  "suggested_fix": { "operation": "replace_node", "target_stable_id": "..." }
}
```

### Symbol and dependency queries

```
query_symbols(module_path) -> SymbolTable
query_dependencies(package) -> DependencyGraph
query_effect_summary(function_stable_id) -> EffectSet
query_borrow_summary(function_stable_id) -> BorrowGraph
query_capability_requirements(function_stable_id) -> CapabilitySet
```

### Canonical formatter (optional in v1)

The formatter is a derived artifact — optional in v1 per REPR-004.
In v1, the canonical typed graph remains the SOLE source of truth.
Source text is derived only and is NEVER normative.
Text representation pipeline (all steps are independent deliverables):

```
print(graph) -> text                  # printer  (REPR-002): graph → text
format(text) -> canonical_text        # formatter (REPR-004): text → canonical text
parse(canonical_text) -> graph        # parser    (REPR-003): text → graph; optional
```

`format(graph)` is NOT a valid call — the formatter operates on text, not on graphs.
The correct call chain: `format(print(graph))`.

If the formatter is provided it MUST be:
```
- deterministic: format(t) depends only on text content and edition
- idempotent:    format(format(t)) == format(t)
- edition-aware: output is valid only for the declared edition
```

Round-trip property `parse(format(print(g)))` is semantically equivalent to `g`
only if BOTH printer AND parser are provided (REPR-003, REPR-005).
Conformance is determined by the canonical graph, not by textual agreement.

### Optimization remarks

```
optimization_remarks(program_graph, optimization_level) -> RemarkSet
  - reports inlining decisions, vectorization, layout choices
  - machine-readable, keyed by stable_id
```

### Example API

```
axiom.compile(program_graph, edition: "axiom-v1")
axiom.incremental_verify(program_graph, patch)
axiom.query_borrow_summary(fn_stable_id)
axiom.query_capability_requirements(fn_stable_id)
axiom.semantic_diff(graph_before, graph_after)

-- optional text pipeline (REPR-002 through REPR-004):
axiom.print(program_graph) -> text              -- printer:   graph → text
axiom.format(text) -> canonical_text            -- formatter: text  → canonical text
axiom.parse(text) -> program_graph              -- parser:    text  → graph (REPR-003; MAY)
-- NOTE: axiom.format(program_graph) is NOT valid; formatter is text→text only
```

---

# Phase 12 — Agent SDK

## Goal

Enable agents to synthesize programs directly.

### Agent capabilities

```
generate program graph
generate patches
verify constraints
compute semantic diff
compile binaries
deploy artifacts
query diagnostics and summaries
```

### Typical workflow

```
agent declares target edition
agent generates graph or patch
agent validates program in strict mode
unknown constructs are rejected
agent queries structured diagnostics and fixes errors
agent queries borrow/effect summaries to verify correctness
agent compiles deterministic native binary
agent deploys artifact
```

---

# Team Requirements

Minimal team:

```
2 compiler engineers
1 type system specialist (ownership/borrow systems experience required)
1 runtime engineer
1 systems engineer
```

Alternative:

```
1 senior engineer + autonomous agents
```

---

# Research Spikes and Kill Criteria

Before committing to full implementation, the following research spikes MUST be completed:

### Spike 1 — stable_id collision resistance (Phase 1–2)

Validate that the stable_id allocation scheme preserves identity across sibling insertion, sibling deletion, and sibling reorder while maintaining acceptable collision resistance under adversarial program graph shapes and large programs. Kill criterion: if stable_id churn occurs for unaffected siblings or collision rate exceeds 1 in 10^12 nodes, the scheme MUST be revised before Phase 3 begins.

### Spike 2 — ownership/borrow inference tractability (Phase 4)

Prototype ownership inference on a representative subset of systems programs. Kill criterion: if inference requires exponential time on realistic programs, the type system design MUST be revised (e.g., require more explicit annotations).

### Spike 3 — region + borrow interaction correctness (Phase 4–5)

Formally verify (or proof-check) the interaction rules between region lifetimes and borrow scopes on a core calculus. Kill criterion: if soundness counterexamples are found, the rules MUST be revised before the full checker is implemented.

### Spike 4 — capability delegation performance (Phase 6–7)

Measure the overhead of passing capability objects through task graphs at realistic depths and fan-out. Kill criterion: if capability passing overhead exceeds 5% of task scheduling overhead, the representation MUST be optimized.

### Spike 5 — backend selection (Phase 8–9)

Build a minimal CPG → SSA → native binary pipeline using both LLVM and Cranelift. Kill criterion: select the backend that achieves acceptable compile-time performance at the end of Phase 8; the unselected backend is deferred to a future phase.

---

# Milestone-Based Timeline

The timeline below is organized around **verifiable milestones** rather than calendar commitments. Each phase has explicit entry criteria (what must be true before starting) and exit criteria (what must be true before the next phase begins).

## Phase 0 — Module/Package/Edition
- Entry: none
- Exit: `spec/modules.yaml` accepted; edition model reviewed

## Phase 1 — Core Specification
- Entry: Phase 0 complete
- Exit: all `spec/*.yaml` files reviewed and internally consistent; no open P0 contradictions

## Phase 2 — Canonical Program Graph
- Entry: Phase 1 complete; Spike 1 passed
- Exit: CPG schema implemented; stable_id and semantic_id tests pass; graph
  serialization (graph → bytes → graph) round-trips losslessly.
  Text pipeline (printer, formatter, parser) is NOT a Phase 2 exit criterion —
  all three are MAY in v1 (REPR-002 through REPR-004). If a parser is provided
  as a later optional deliverable, parse(print(g)) MUST produce a graph
  semantically equivalent to g; that check is conditional on parser presence.

## Phase 3 — Patch System
- Entry: Phase 2 complete
- Exit: `apply_patch`, `incremental_validate`, `semantic_diff` pass test suite including replace_node stable_id preservation tests

## Phase 4 — Type System (includes ownership/borrow)
- Entry: Phase 2 complete; Spike 2 passed; Spike 3 passed
- Exit: `axiom-typecheck` passes ownership/borrow/move/alias test suite; no known soundness gaps

## Phase 5 — Region Memory System
- Entry: Phase 4 complete
- Exit: region validator integrated with Phase 4 checker; use-after-region and cross-region alias tests pass

## Phase 6 — Effect System and Capabilities
- Entry: Phase 4 complete; Spike 4 passed
- Exit: effect analyzer with capability flow checker passes test suite; ambient authority tests rejected

## Phase 7 — Concurrency Model
- Entry: Phases 5 and 6 complete
- Exit: CON-001 through CON-011 validated; region ownership transfer tests pass; failure propagation tests pass

## Phase 8 — Intermediate Representation
- Entry: Phase 7 complete
- Exit: CPG → SSA lowering complete; SSA IR schema and invariants validated; Spike 5 complete; backend selected

## Phase 9 — Backend
- Entry: Phase 8 complete
- Exit: `axiom-compiler` produces correct native binaries for initial language subset

## Phase 10 — Minimal Runtime
- Entry: Phase 9 complete
- Exit: `axiom-runtime` supports region allocator, task scheduler, root capability constructors

## Phase 11a — Agent Tooling (graph-level; no backend dependency)
- Entry: Phase 7 complete  ← runs in parallel with Phases 8–10; does NOT wait for runtime
- Rationale: diagnostics, graph queries, borrow/effect/capability summaries, semantic_diff,
  and incremental_validate all operate on the CPG and validation results. None require a
  compiled binary or a running runtime. Unlocking this track early creates the most
  valuable feedback loop for language design.
- Exit: the following APIs are implemented, tested, and documented:
  - structured diagnostics (JSON, keyed by stable_id, with requirement ID + suggested patch)
  - `query_symbols(module_path) -> SymbolTable`
  - `query_effect_summary(function_stable_id) -> EffectSet`
  - `query_borrow_summary(function_stable_id) -> BorrowGraph`
  - `query_capability_requirements(function_stable_id) -> CapabilitySet`
  - `semantic_diff(graph_before, graph_after) -> DiffReport`
  - `incremental_verify(graph, patch) -> ValidationResult`
  - optional: `print(graph) -> text`, `format(text) -> canonical_text` (REPR-002, REPR-004)

## Phase 11b — Agent Tooling (compilation API; requires backend + runtime)
- Entry: Phases 9 and 10 complete AND Phase 11a complete
- Exit: the following APIs are implemented, tested, and documented:
  - `axiom.compile(program_graph, edition) -> NativeBinary`
  - `optimization_remarks(program_graph, optimization_level) -> RemarkSet`
  - end-to-end compile + run smoke test passes for the initial language subset

## Phase 12 — Agent SDK
- Entry: Phases 11a and 11b complete
- Exit: end-to-end agent synthesis workflow passes integration test suite

---

# Critical Component

The **Canonical Program Graph + Patch Engine + Validation Engine** is the most critical component.

All other systems depend on it. The stable_id derivation algorithm is the single point of failure for semantic_diff and incremental verification stability.

---

# First Achievable Milestone

Create a system where an agent can:

```
1 create a program graph (with stable_ids)
2 apply structural patches (with stable_id preservation)
3 run incremental validation
4 compute semantic diff (keyed by stable_id)
5 compile to a native binary
```

Initial language subset:

```
functions
records
pattern matching
regions (with ownership/borrow rules)
Outcome<T,E>
explicit effects with capability objects
module declarations
```

---

# Expected Outcome

AXIOM becomes the first programming language designed primarily for **AI synthesis rather than human coding**, with formal guarantees on memory safety, data-race freedom, capability-bounded authority, and deterministic reproducible compilation.
