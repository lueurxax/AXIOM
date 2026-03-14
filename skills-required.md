# Required Skills for AXIOM Implementation

Список скилов, необходимых для реализации языка AXIOM по 12 фазам.
Каждый скил привязан к фазам и требованиям из спецификации.

---

## Phase 0 — Module, Package, and Edition Semantics

Phase 0 является обязательным фундаментом — Phase 1 не может начаться до его завершения.

| Скил | Описание | Треб. |
|------|----------|-------|
| **Module system design** | Проектирование Module как first-class compilation unit: explicit exports/imports, visibility (public/internal/private), deterministic name resolution, no import side effects, no cyclic imports | MOD-001, MOD-002, MOD-005, MOD-SCOPE-001–006 |
| **Edition model** | Механизм editions как named immutable snapshot языковой семантики; gating breaking changes; machine-detectable deprecation paths; агенты обязаны декларировать target edition | MOD-004, P15 |
| **YAML spec authoring (modules)** | Написание `spec/modules.yaml` как нормативного документа Phase 0; выход Phase 0 — принятый `spec/modules.yaml` | — |

---

## Phase 1 — Core Specification

| Скил | Описание | Треб. |
|------|----------|-------|
| **Spec authoring** | Написание machine-consumable YAML-спецификаций: types, effects, errors, memory, concurrency, abi, determinism, validation, ir, modules | — |
| **Formal grammar design** | Описание грамматики языка (BNF/EBNF или PEG); P9 требует low-ambiguity синтаксис. **Опционально в v1** — парсер текстового синтаксиса является derived-артефактом и не обязателен (REPR-003). Грамматика нужна только если реализуется текстовый frontend. | P9, REPR-003 |
| **JSON/YAML schema design** | Проектирование схем для программного графа, патчей, диагностик | ARCH-001 |

---

## Phase 2 — Canonical Program Graph

| Скил | Описание | Треб. |
|------|----------|-------|
| **Typed directed graph data structures** | Узлы, рёбра, stable node identity; типы рёбер: control_flow, data_flow, effect_flow, ownership_flow, task_dependency | ARCH-001, ARCH-002 |
| **Graph canonicalization** | Алгоритм нормализации графа к уникальной форме (canonical form → unique hash) | ARCH-001 |
| **stable_id / semantic_id derivation** | Реализация двух хешей: stable_id (path-anchored) для named и anonymous nodes (structural_discriminator); semantic_id (content-anchored) для moved_nodes detection в semantic_diff | ARCH-001, Spike 1 |
| **Graph serialization** | Бинарная и JSON сериализация/десериализация CPG | ARCH-002 |

---

## Phase 3 — Patch System

| Скил | Описание | Треб. |
|------|----------|-------|
| **Patch engine** | Реализация примитивов: `insert_node`, `replace_node`, `reconnect_edges`, `modify_contract`; stable_id preservation rules под каждую операцию | ARCH-003 |
| **Structural semantic diff** | Вычисление семантических различий между двумя состояниями CPG; классификация added/removed/modified/moved nodes по stable_id и semantic_id | ANA-002 |
| **Incremental validation** | Валидация только затронутых патчем узлов и рёбер | ANA-001 |

---

## Phase 4 — Type System (includes ownership/borrow checker)

Phase 4 требует прохождения **Spike 2** (ownership inference tractability) и **Spike 3** (region+borrow interaction correctness) до начала реализации.

| Скил | Описание | Треб. |
|------|----------|-------|
| **Type inference** | Детерминированный алгоритм вывода типов (Hindley-Milner или bidirectional) | TYPE-001 |
| **Algebraic data types & generics** | ADT (sum/product types), параметрический полиморфизм | TYPE-001 |
| **Exhaustive pattern matching** | Проверка полноты match; reject при missing_variants и unreachable_branches | TYPE-002 |
| **`Outcome<T,E>` type** | Единственный fallible return type; запрет Result, exceptions, null errors, sentinel errors; `Outcome<T,Void>` запрещён для моделирования отсутствия | ERR-001, ERR-SCOPE-001–002 |
| **Ownership and move semantics** | Режимы владения: owned/borrowed_shared/borrowed_exclusive/moved; enforcement правил MOVE-001, MOVE-002 на уровне type checker | MEM-SCOPE-003, MOVE-001, MOVE-002 |
| **Borrow checker** | Enforcement BORROW-001, BORROW-002 (нет coexisting exclusive+shared; reference не outlive region); ALIAS-001, ALIAS-002 (alias safety); RACE-001 (cross-task access requires disjointness proof or sync primitive). Не опциональен в safe code. | BORROW-001, BORROW-002, ALIAS-001, ALIAS-002, RACE-001, MEM-SCOPE-006 |
| **Unsafe block contracts** | Механизм escape hatch для unsafe кода с explicit, auditable contract; borrow rules overridable только в unsafe | P3, P4 |

---

## Phase 5 — Region Memory System

Phase 5 реализует allocation/deallocation discipline поверх ownership discipline из Phase 4. Оба компонента требуются для memory safety — ни один не достаточен сам по себе.

| Скил | Описание | Треб. |
|------|----------|-------|
| **Region/arena allocator design** | Статическая модель регионов: explicit allocation scope, lifetime; объект принадлежит ровно одному региону | MEM-001, MEM-002 |
| **Region lifetime analysis** | Статическая верификация: region destroyed → все объекты в регионе invalid; region не outlive своего declared scope | MEM-001 |
| **Region+borrow integration** | Интеграция region validator с borrow checker из Phase 4: use-after-region, cross-region exclusive alias, region-escape | MEM-SCOPE-004, BORROW-002, Spike 3 |
| **Immutability enforcement** | Значения immutable by default; мутация требует явной аннотации | MEM-003, MEM-SCOPE-001, MEM-SCOPE-002 |

---

## Phase 6 — Effect System

Phase 6 требует прохождения **Spike 4** (capability delegation performance) до полной реализации.

| Скил | Описание | Треб. |
|------|----------|-------|
| **Effect inference & propagation** | Распространение эффектов по call graph; effects propagate transitively | EFF-001 |
| **Effect graph construction** | Построение графа зависимостей эффектов для capability enforcement | EFF-002 |
| **Capability model enforcement** | Доступ к ресурсам только через explicit capability objects (not ambient); CAP-001 – CAP-005; authority boundaries: split/revoke/delegation; capability mockability для тестирования | EFF-003, CAP-001–005 |
| **Strict effect validation** | Отклонение unknown effects в strict mode; `reject_unknown_constructs: true`; агенты обязаны декларировать edition | EFF-001, MOD-004 |

---

## Phase 7 — Concurrency Model

| Скил | Описание | Треб. |
|------|----------|-------|
| **Structured concurrency model** | Parent-child task tree (CON-001); parent MUST NOT complete while children running (CON-002); propagating cancellation; explicit join/timeout | CON-001, CON-002 |
| **Task dependency graph** | Построение и валидация task graph; parent_child edges MUST form a tree (CON-001: каждая задача имеет ровно одного родителя, кроме root); dependency edges образуют отдельную структуру и не являются частью parent-child дерева; validator enforces CON-001 – CON-010 | CON-001–010 |
| **Operational semantics: join/await** | `join`, `await_all` (fail-fast и collect_errors mode), `await_any` (all-failed/all-cancelled/empty-handles cases); `with_timeout` | CON-003–006 |
| **Region ownership transfer** | Передача owned Region между parent/child tasks; статическая верификация CON-007 – CON-010 | CON-007–010 |
| **Data race analysis** | Статическое доказательство отсутствия гонок в safe code (RACE-001) через task disjointness или sync primitives | RACE-001 |

---

## Phase 8 — Intermediate Representation

Phase 8 включает выполнение **Spike 5** (backend selection: LLVM vs Cranelift).

| Скил | Описание | Треб. |
|------|----------|-------|
| **SSA construction** | Direct lowering CPG → SSA; phi-node placement, dominator tree. HIR is explicitly out of scope in v1 (plan Phase 8); the normative pipeline is CPG → SSA → native. HIR MAY be introduced in a future edition if the CPG→SSA gap proves too large. | COMP-001 |
| **IR design** | Проектирование `axiom-ir` с сохранением инвариантов типов, эффектов, регионов, ownership mode | COMP-001 |
| **IR optimizations** | Базовые SSA-оптимизации: DCE, LICM, inlining, constant folding | PERF-001 |

---

## Phase 9 — Backend

Backend выбирается по итогам **Spike 5**: реализуются оба прототипа (LLVM и Cranelift), затем выбирается один. Неотобранный backend откладывается на будущую фазу.

| Скил | Описание | Треб. |
|------|----------|-------|
| **LLVM integration** | Генерация LLVM IR, target triple, ABI mapping, оптимизационные passes. Требуется для Spike 5 прототипа. | COMP-002 |
| **Cranelift integration** | Быстрый backend (cranelift-codegen): быстрое compile time, простая интеграция. Требуется для Spike 5 прототипа. | COMP-002 |
| **Backend selection (Spike 5)** | Сравнение LLVM и Cranelift по compile-time performance; выбор одного; документирование kill criterion | Spike 5 |
| **C ABI interop** | Декларация layout, calling convention, ownership transfer на FFI границе | ABI-001 |
| **Deterministic code generation** | Воспроизводимые бинарные выходы; layout determinism | DET-001, PERF-002 |

---

## Phase 10 — Minimal Runtime

| Скил | Описание | Треб. |
|------|----------|-------|
| **Runtime region allocator** | Run-time реализация arena allocator без GC | PERF-003 |
| **Minimal task scheduler** | Cooperative/structured async-рантайм без green threads | CON-001, PERF-003 |
| **Root capability constructors** | Рантайм предоставляет ровно один конструктор на каждый non-structural эффект (CAP-006); доступны только в root entrypoint, не в library code (CAP-003): `acquire_fs_read_cap(path_prefix) -> FsReadCap`, `acquire_fs_write_cap(path_prefix) -> FsWriteCap`, `acquire_network_read_cap(host_allow_list) -> NetworkReadCap`, `acquire_network_write_cap(host_allow_list) -> NetworkWriteCap`, `acquire_database_read_cap(connection_spec) -> DatabaseReadCap`, `acquire_database_write_cap(connection_spec) -> DatabaseWriteCap`, `acquire_time_cap() -> TimeCap`, `acquire_random_cap() -> RandomCap`, `acquire_process_cap(allowed_commands) -> ProcessCap`, `acquire_unsafe_ffi_cap(allowed_symbols) -> UnsafeFfiCap`. `region_alloc` и `pure` не имеют конструкторов — они структурные. | EFF-003, CAP-003, CAP-006 |
| **System interface / syscall layer** | Minimal syscall bindings; запрет большого libc-рантайма; запрет ambient authority | PERF-003 |

---

## Phase 11 — Agent Tooling

| Скил | Описание | Треб. |
|------|----------|-------|
| **Compiler API design** | Library/RPC API: `parse_graph`, `validate_graph`, `apply_patch`, `semantic_diff`, `incremental_verify`, `typecheck`, `lower_to_ssa`, `compile` | P11 |
| **Structured JSON diagnostics** | Машиночитаемые диагностики: diagnostic_id, severity, requirement (e.g. BORROW-001), node_stable_id, suggested_fix; не free-form text | P11, tooling_contract |
| **Incremental compilation API** | Поддержка инкрементальных запросов; кэш валидации | ANA-001 |
| **Query and summary APIs** | `query_symbols`, `query_dependencies`, `query_effect_summary`, `query_borrow_summary`, `query_capability_requirements`; без этих API Phase 12 agent workflow невозможен | P11, ANA-001, ANA-002 |
| **Canonical formatter (optional)** | Formatter operates on **text, not graphs**. Correct pipeline: `print(graph) -> text` (printer, REPR-002), then `format(text) -> canonical_text` (formatter, REPR-004), then optionally `parse(canonical_text) -> graph` (parser, REPR-003). `format(graph)` is NOT a valid call. Formatter MUST be deterministic, idempotent, and edition-aware. Semantics-preserving: if a parser is also provided, `parse(format(print(g)))` MUST produce a graph semantically equivalent to `g` (REPR-005). является derived artifact, не normative. | REPR-003, REPR-004, REPR-005 |
| **Optimization remarks API** | `optimization_remarks(graph, level) -> RemarkSet`; keyed by stable_id; machine-readable | PERF-001 |

---

## Phase 12 — Agent SDK

| Скил | Описание | Треб. |
|------|----------|-------|
| **Agent synthesis workflow** | Generate graph → patch → validate → compile → deploy pipeline; агент декларирует edition, валидирует в strict mode, исправляет ошибки по structured diagnostics | MOD-004, ANA-001, ANA-002, COMP-002 |
| **Edition/version targeting** | Механизм editions для стабильного targeting агентами; agents MUST declare `axiom-v1` | MOD-004, P15 |
| **Strict mode enforcement** | `reject_unknown_constructs: true`; аудит capability boundaries; все features MUST satisfy spec requirements | EFF-003, MOD-004, P3 |

---

## Research Spikes (обязательные, gating реализацию)

Спайки — это kill-criterion-gated исследования. Фаза не может начаться до прохождения ведущих её спайков.

| Spike | Гейтирует | Скил | Kill criterion |
|-------|-----------|------|----------------|
| **Spike 1 — stable_id collision resistance** | Phase 2 | Криптографическое хеширование, empirical collision analysis | Collision rate > 1 in 10¹² → хеш-схема MUST быть пересмотрена |
| **Spike 2 — ownership/borrow inference tractability** | Phase 4 | Type systems research, ownership inference prototyping | Inference требует exponential time на realistic programs → type system design MUST быть пересмотрен |
| **Spike 3 — region+borrow interaction correctness** | Phase 4–5 | Formal methods, proof checking (Coq/Lean/TLA+), core calculus | Найдены soundness counterexamples → правила взаимодействия MUST быть исправлены до реализации |
| **Spike 4 — capability delegation performance** | Phase 6–7 | Performance engineering, micro-benchmarking task graphs | Overhead capability passing > 5% task scheduling overhead → representation MUST быть оптимизирована |
| **Spike 5 — backend selection** | Phase 8–9 | LLVM + Cranelift prototyping, compile-time benchmarking | Оба backend прототипируются; выбирается один по compile-time; второй откладывается |

---

## Сквозные скилы (все фазы)

| Скил | Применение |
|------|------------|
| **Property-based testing** | Верификация инвариантов CPG, типовой системы, эффектов, патчей |
| **Fuzzing** | Поиск нарушений безопасности памяти и инвариантов (P4) |
| **Benchmarking** | Проверка zero-cost abstractions и детерминизма (PERF-001) |
| **Reproducible builds** | DET-001 — детерминированная компиляция на CI/CD |
| **Spec conformance testing** | Автоматическая проверка соответствия реализации YAML-спецификации |
| **Structured logging / observability** | P14 — telemetry, tracing, structured panic reports |
| **Formal methods** | Spike 3 (обязателен); рекомендован для Phase 4–5 soundness verification |

---

## Язык реализации компилятора

Выбор языка для самого компилятора — отдельное архитектурное решение:

| Вариант | Плюсы | Минусы |
|---------|-------|--------|
| **Rust** | Ownership/lifetimes коррелируют с семантикой AXIOM; inkwell (LLVM), cranelift-codegen | Высокий порог входа |
| **Go** | Простота, хорошая поддержка графов, быстрая итерация | Нет ownership model; GC |
| **C++** | Прямая интеграция с LLVM API | Небезопасный, сложный |

Rust является предпочтительным вариантом: семантика языка-реализации максимально близка к семантике AXIOM (ownership, regions, explicit effects).
