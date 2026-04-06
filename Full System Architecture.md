# Full System Architecture

> **Status**: Complete closed specification — all decisions applied, no TBDs.
> **Revision**: 1.0 — 2026-04-06

---

## Table of Contents

1. [Vision and Problem Statement](#1-vision-and-problem-statement)
2. [Architectural Decisions Log](#2-architectural-decisions-log)
3. [Core Data Model](#3-core-data-model)
4. [Seven-Layer Architecture](#4-seven-layer-architecture)
5. [Graph Engine](#5-graph-engine)
6. [Computation Layer](#6-computation-layer)
7. [Query Language — Text DSL and Visual Builder](#7-query-language--text-dsl-and-visual-builder)
8. [Range Query Index](#8-range-query-index)
9. [Validation System](#9-validation-system)
10. [Multi-Supertag Merge and Precedence](#10-multi-supertag-merge-and-precedence)
11. [Sync and Conflict Resolution](#11-sync-and-conflict-resolution)
12. [Offline Parity Model](#12-offline-parity-model)
13. [Storage Layer](#13-storage-layer)
14. [Frontend Architecture](#14-frontend-architecture)
15. [AI and Automation Layer](#15-ai-and-automation-layer)
16. [API Reference](#16-api-reference)
17. [Technology Stack](#17-technology-stack)
18. [Security and Multi-Tenancy](#18-security-and-multi-tenancy)
19. [Deployment and Operations](#19-deployment-and-operations)

---

## 1. Vision and Problem Statement

### 1.1 Motivation

Two categories of tools have historically served distinct user needs:

| Dimension | Graph/Notes tools (e.g., Tana) | Relational/Table tools (e.g., Grist) |
|---|---|---|
| Schema | Schema-on-read | Schema-on-write |
| Conflict resolution | CRDT, eventually consistent | Push DAG, deterministic |
| Validation | Loose, opt-in | Strict, mandatory |
| Primary abstraction | Node/graph, free-form | Row/table, typed columns |
| Query model | Tag traversal, fuzzy | SQL, precise |

These two paradigms are converging in practice: users want the fluidity of a note graph and the rigor of a database inside the same workspace. Existing tools force a hard choice.

### 1.2 Core Thesis

> **"The node is universal; the validation policy is contextual."**

Every piece of information — a note, a task, a database record, a formula, a query, an automation — is the same underlying primitive: a **node** in a directed labeled multigraph. What differentiates "note" from "database record" is not a different data type; it is a different **validation policy** applied at runtime.

This architecture delivers a unified primitive capable of behaving as:

- A free-form knowledge graph (Tana-style, schema-on-read, CRDT, no enforcement).
- A strictly-typed relational database (Grist-style, schema-on-write, push DAG, mandatory validation).
- Any point on the continuum between those extremes.

### 1.3 Scope

This document covers the full system: data model, computation, storage, sync, frontend, offline operation, AI/automation, API, security, and deployment.

---

## 2. Architectural Decisions Log

The following decisions close all open design questions. Every section of this document reflects these decisions without further qualification.

| # | Question | Decision | Rationale |
|---|---|---|---|
| **D1** | Query language UX | **C — Both text DSL and visual query builder; bidirectional round-trip (visual ↔ DSL)** | Power users get DSL speed; less technical users get visual builder; both stay in sync. |
| **D2** | Multi-supertag field conflict | **C — Prompt user to resolve; store per-field resolution persistently** | Silently picking a winner destroys data expectations; user intent is the ground truth. |
| **D3** | Range queries | **C — Add per-field sorted index (BTree / sorted Vec) for number and date fields** | Hash indexes cannot serve range predicates efficiently; sorted indexes fill the gap. |
| **D4** | Conflict markers | **A — Store as node metadata in graph; replicate via CRDT as a boolean flag** | Conflict state is first-class information, not ephemeral UI state; must survive reconnection. |
| **D5** | Offline parity | **B — Partial offline: validation + queries run locally; sandbox formula evaluation is server-only** | Full offline formula sandboxing requires shipping a full Extism/WASM runtime to every client, increasing bundle size and attack surface unacceptably. |

---

## 3. Core Data Model

### 3.1 Node

Every entity in the system is a **Node**:

```
Node {
  id:           NodeId,           // globally unique, 128-bit UUID v7 (time-ordered)
  content:      NodeContent,      // rich text, scalar value, or structured payload
  supertags:    Vec<TagId>,       // applied class constructors (ordered, see §10)
  metadata:     NodeMetadata,     // timestamps, author, revision vector, conflict flags
  schema_state: SchemaState,      // Open | Guarded | Strict
}
```

`NodeId` is a 128-bit UUID v7 (time-ordered). All node creation events embed the originating `AgentId` (user or automation) and a Hybrid Logical Clock (HLC) timestamp.

**NodeMetadata** carries:

```
NodeMetadata {
  created_at:       HLCTimestamp,
  updated_at:       HLCTimestamp,
  author_id:        AgentId,
  revision:         u64,
  conflict_marker:  Option<ConflictMarker>,   // D4: first-class field
  crdt_vector:      VectorClock,
}
```

`ConflictMarker` (Decision D4):

```
ConflictMarker {
  is_conflicted:    bool,         // replicated as CRDT flag
  conflicting_ops:  Vec<OpId>,    // IDs of the operations in conflict
  resolution:       Option<Resolution>,  // None = unresolved
}
```

### 3.2 Six Typed Edge Classes

All relationships are typed directed edges. There are exactly six edge classes, each stored in a separate adjacency partition for O(1) filtered traversal:

| Symbol | Name | Semantics |
|---|---|---|
| **E_H** | Hierarchy | Parent → child structural containment (the outline tree) |
| **E_S** | Semantic / Supertag | Node → supertag definition; "this node is an instance of class T" |
| **E_A** | Attribute / Field | Node → field value; "this node has field F with value V" |
| **E_R** | Reference | Node → node; arbitrary cross-reference (does not imply ownership) |
| **E_D** | Dependency / Formula | Formula node → its input nodes; drives the push computation DAG |
| **E_I** | Integrity / Referential | Node → node; enforced referential constraint (cascade, restrict, or nullify on delete) |

Each edge carries:

```
Edge {
  id:       EdgeId,
  kind:     EdgeKind,        // E_H | E_S | E_A | E_R | E_D | E_I
  source:   NodeId,
  target:   NodeId,
  metadata: EdgeMetadata,    // weight, label, created_at, author_id
}
```

### 3.3 Supertags as Class Constructors

A **Supertag** is a node whose content defines a schema — a set of field definitions, each with:

```
FieldDefinition {
  field_id:    FieldId,
  name:        String,
  value_type:  ValueType,       // Text | Number | Date | Bool | NodeRef | Enum(Vec<String>)
  policy:      ValidationPolicy, // Open | Guarded | Strict
  default:     Option<Value>,
  required:    bool,
}
```

When a supertag is applied to a node (via an E_S edge), the Validation Layer materialises the expected field set for that node. Schema-on-read is preserved: applying a supertag never deletes existing data — it adds expectations.

### 3.4 Value Types

```
ValueType =
  | Text(TextOptions)
  | Number(NumberOptions)    // integer or float; participates in range index (D3)
  | Date(DateOptions)        // ISO-8601; participates in range index (D3)
  | Bool
  | NodeRef(Option<TagId>)   // typed or untyped reference
  | Enum(Vec<String>)
  | Formula(FormulaExpr)     // computed; not stored directly
  | FileRef
```

---

## 4. Seven-Layer Architecture

The system is organized into seven horizontal layers plus one cross-cutting layer. Data flows vertically; the cross-cutting AI/Automation layer injects at any level.

```
┌─────────────────────────────────────────────────────────────────────┐
│  PRESENTATION LAYER                                                 │
│  SolidJS · ProseMirror/TipTap · View renderers                     │
│  Outliner · Table · Kanban · Calendar · Chart · Query builder UI   │
├─────────────────────────────────────────────────────────────────────┤
│  VALIDATION LAYER                                                   │
│  Field policy enforcement · Supertag merge resolution (D2)         │
│  Schema-on-read / schema-on-write boundary                         │
├─────────────────────────────────────────────────────────────────────┤
│  COMPUTATION LAYER                                                  │
│  Push formula DAG (E_D edges) · Pull live query subscriptions      │
│  Query DSL parser · Visual query builder (D1) · Range index (D3)  │
├─────────────────────────────────────────────────────────────────────┤
│  SEMANTIC LAYER                                                     │
│  Supertag application · Field resolution · Inverted tag/field index│
├─────────────────────────────────────────────────────────────────────┤
│  GRAPH RUNTIME LAYER                                                │
│  In-memory node/edge store · Dependency DAG · Traversal API        │
├─────────────────────────────────────────────────────────────────────┤
│  SYNC LAYER                                                         │
│  Yjs CRDT · Conflict detection · Conflict markers (D4)             │
│  Offline queue · Partial-offline boundary (D5)                     │
├─────────────────────────────────────────────────────────────────────┤
│  PERSISTENCE LAYER                                                  │
│  Event log (PostgreSQL / SQLite) · DuckDB relational projection    │
│  Snapshot management                                                │
└─────────────────────────────────────────────────────────────────────┘
         ↕ (cuts across all layers)
┌─────────────────────────────────────────────────────────────────────┐
│  AI / AUTOMATION LAYER                                              │
│  LiteLLM · MCP · Extism plugin sandbox · Automation triggers       │
└─────────────────────────────────────────────────────────────────────┘
```

Layer responsibilities are exclusive: no layer calls downward by more than one level except via defined interfaces.

---

## 5. Graph Engine

### 5.1 Purpose

The **Graph Runtime Layer** is the primary in-memory data substrate. It is derived entirely from the event log by replay; it is never the source of truth. Its sole purpose is sub-millisecond traversal and index lookup during formula evaluation and query execution.

### 5.2 Same-Binary Principle

All graph engine code lives in a single Rust library crate: **`graph-engine-core`**. Conditional compilation (`cfg` feature flags) produces two targets:

| Target | Feature flag | Additional capabilities |
|---|---|---|
| Server (native) | `server` | OS threading, event log integration, Prometheus metrics, DuckDB integration |
| Client (WASM) | `client` | `wasm-bindgen` exports, JS-friendly serialization, no threading |

This guarantees identical traversal semantics and query results between client and server, which is the prerequisite for Decision D5 (partial offline).

### 5.3 Data Structures

#### 5.3.1 Node Store

```rust
struct NodeStore {
    nodes: HashMap<NodeId, NodeData>,   // O(1) lookup by id
}

struct NodeData {
    id:           NodeId,
    content:      NodeContent,
    supertags:    SmallVec<[TagId; 4]>,
    metadata:     NodeMetadata,
    schema_state: SchemaState,
}
```

#### 5.3.2 Edge Store

Six adjacency lists, one per edge class, each stored as:

```rust
struct EdgePartition {
    outgoing: HashMap<NodeId, SmallVec<[EdgeId; 8]>>,
    incoming: HashMap<NodeId, SmallVec<[EdgeId; 8]>>,
    edges:    HashMap<EdgeId, Edge>,
}

struct EdgeStore {
    hierarchy:    EdgePartition,   // E_H
    semantic:     EdgePartition,   // E_S
    attribute:    EdgePartition,   // E_A
    reference:    EdgePartition,   // E_R
    dependency:   EdgePartition,   // E_D
    integrity:    EdgePartition,   // E_I
}
```

#### 5.3.3 Inverted Indexes (Semantic Layer)

Two inverted indexes support tag-membership and field-value lookups:

```rust
struct TagIndex {
    tag_to_nodes: HashMap<TagId, HashSet<NodeId>>,
}

struct FieldValueIndex {
    // Exact match: (field, value) → nodes
    exact: HashMap<(FieldId, FieldValue), HashSet<NodeId>>,
}
```

#### 5.3.4 Range Index (Decision D3)

For every field of type `Number` or `Date`, a **per-field sorted index** is maintained:

```rust
struct RangeIndex {
    // BTreeMap keeps keys in sorted order for O(log n) range scan
    numeric: HashMap<FieldId, BTreeMap<OrderedFloat<f64>, HashSet<NodeId>>>,
    date:    HashMap<FieldId, BTreeMap<NaiveDate, HashSet<NodeId>>>,
}
```

**Range scan algorithm**:

```
range_query(field_id, lo, hi) → HashSet<NodeId>:
  tree ← range_index[field_id]
  result ← ∅
  for (key, node_set) in tree.range(lo..=hi):
    result ← result ∪ node_set
  return result
```

This delivers O(log n + k) range scans where k is the number of matching nodes, versus O(n) full-scan without the index.

**Index maintenance**: On every `ApplyMutation` that creates, updates, or deletes an attribute edge (E_A) for a Number or Date field, the range index is updated atomically within the same mutation transaction.

#### 5.3.5 Dependency DAG

```rust
struct DependencyDAG {
    // E_D edges only; must remain acyclic
    edges:            EdgePartition,
    topological_order: Vec<NodeId>,   // incrementally maintained
    dirty_nodes:      HashSet<NodeId>, // nodes needing re-evaluation
}
```

Cycle prevention is enforced at edge creation time using incremental topological sort (Pearce-Kelly algorithm). Any `add_dependency(source, target)` call that would introduce a cycle returns `Err(CycleError)` and the mutation is rejected.

### 5.4 Graph Read API

```rust
trait GraphReadAPI {
    fn get_node(&self, id: NodeId) -> Option<&NodeData>;
    fn get_edges(&self, node: NodeId, kind: EdgeKind, direction: Direction)
        -> impl Iterator<Item = &Edge>;
    fn query_tag_index(&self, tag: TagId) -> &HashSet<NodeId>;
    fn query_field_exact(&self, field: FieldId, value: &FieldValue)
        -> &HashSet<NodeId>;
    fn query_field_range(&self, field: FieldId, lo: &Value, hi: &Value)
        -> HashSet<NodeId>;    // D3: delegates to RangeIndex
    fn get_dependency_order(&self) -> &[NodeId];   // topological
}
```

### 5.5 Graph Mutate API

```rust
trait GraphMutateAPI {
    fn apply_mutation(&mut self, mutation: Mutation) -> Result<MutationResult, MutationError>;
}

enum Mutation {
    CreateNode(CreateNodeParams),
    UpdateNodeContent(NodeId, NodeContent),
    DeleteNode(NodeId),
    AddEdge(AddEdgeParams),
    RemoveEdge(EdgeId),
    ApplySupertag(NodeId, TagId),        // triggers validation + supertag merge if needed
    SetFieldValue(NodeId, FieldId, Value),
    SetConflictMarker(NodeId, ConflictMarker),  // D4
}

struct MutationResult {
    affected_nodes:   Vec<NodeId>,
    dirty_formulas:   Vec<NodeId>,       // nodes that need re-evaluation
    supertag_prompts: Vec<MergePrompt>,  // D2: fields needing user resolution
}
```

### 5.6 Event Sourcing Contract

The Graph Runtime Layer **never** writes to the event log. It receives `Event` structs from the Persistence Layer and applies them idempotently. The full graph state can always be reconstructed by replaying all events from the log.

---

## 6. Computation Layer

### 6.1 Overview

The Computation Layer implements two complementary computation models that coexist and interoperate:

| Model | Direction | Trigger | Output |
|---|---|---|---|
| **Push formula DAG** | Downward through E_D edges | Node mutation marks source dirty | Formula node values, reactively propagated |
| **Pull live queries** | Outward through indexes | Query subscription registered | Incrementally-updated result sets |

These models are **bidirectional**:
- Formula outputs are indexed by the Semantic Layer → discoverable by queries.
- Query result sets can be passed as inputs to aggregating formulas.
- To prevent infinite loops: query result sets are **read-only snapshots** during a formula evaluation pass; a formula cannot trigger a new query re-evaluation in the same pass.

### 6.2 Push Formula DAG

#### 6.2.1 Formula Node

A formula node is any node whose field value is of type `Formula(FormulaExpr)`. Formulas are expressed in a sandboxed expression language (see §6.2.2) and declare their inputs via E_D edges.

#### 6.2.2 Formula Expression Language

Formulas use a Lisp-like expression language compiled to Extism WASM plugins. This allows:
- Server: full sandbox with resource limits (CPU, memory, I/O).
- Client (offline): not executed locally (Decision D5); a placeholder value is displayed with a "server required" indicator.

```
expr ::=
  | literal
  | field_ref(node_id, field_id)
  | arithmetic(op, expr, expr)
  | comparison(op, expr, expr)
  | if(expr, expr, expr)
  | aggregate(agg_fn, query_result)
  | call_builtin(name, Vec<expr>)

agg_fn ::= SUM | AVG | MIN | MAX | COUNT | CONCAT
```

#### 6.2.3 Incremental Push Evaluation

```
on_mutation(mutation):
  1. Apply mutation to Graph Runtime Layer
  2. For each affected node N in mutation.affected_nodes:
       mark_dirty(N) in DependencyDAG
  3. Compute evaluation_order ← topological_sort(dirty_nodes)
  4. For each formula_node F in evaluation_order:
       inputs ← [resolve_value(dep) for dep in get_deps(F)]
       new_value ← evaluate_formula(F.formula, inputs)   // server-side only (D5)
       if new_value ≠ F.cached_value:
         update_field(F, new_value)
         propagate dirty to F's dependents
  5. Clear dirty set
```

Evaluation is synchronous on the server. On the client, step 4 is skipped; stale cached values are displayed with a "pending sync" indicator.

### 6.3 Pull Live Queries

#### 6.3.1 Query Subscription Lifecycle

```
register_subscription(query, callback):
  1. Parse query (DSL text or visual IR — see §7)
  2. Compile to QueryPlan
  3. Execute QueryPlan against current graph state → initial ResultSet
  4. Register interest sets: which tags, fields, node IDs this query depends on
  5. Return Subscription { id, initial_result }

on_graph_change(affected_nodes, affected_edges):
  For each Subscription S:
    If affected_nodes ∩ S.interest_set ≠ ∅:
      re_execute(S.query) → new_result
      If new_result ≠ S.last_result:
        invoke S.callback(delta(S.last_result, new_result))
        S.last_result ← new_result
```

#### 6.3.2 Materialized Views for Strict Nodes

For node sets tagged with a Strict-policy supertag, the Computation Layer maintains a **fully materialized incremental view** in DuckDB (Persistence Layer). These views are updated synchronously on mutation and can be queried via SQL by external integrations.

### 6.4 Offline Computation Behavior (Decision D5)

| Capability | Online | Offline |
|---|---|---|
| Field validation (Open/Guarded/Strict) | ✅ Full enforcement | ✅ Full enforcement (graph engine in WASM) |
| Query execution (tag, field, range) | ✅ Full | ✅ Full (graph engine in WASM, D3 range index) |
| Formula evaluation (sandboxed Extism) | ✅ Full | ⚠️ Deferred — cached value shown, "pending sync" indicator |
| DuckDB materialized views | ✅ Full | ❌ Not available |
| AI / LLM calls | ✅ Full | ❌ Not available |

When a formula node's value is stale offline, the UI renders it in a distinct style and queues the evaluation request. On reconnection, all pending evaluations are sent to the server in dependency order.

---

## 7. Query Language — Text DSL and Visual Builder

*(Decision D1: both modes, full round-trip)*

### 7.1 Design Principles

1. Every query expressible in the visual builder is expressible as DSL text, and vice versa.
2. Parsing DSL → Visual IR is lossless and deterministic.
3. Editing in the visual builder → DSL round-trip is lossless (comments are stripped; formatting is canonical).
4. The visual builder never generates DSL that cannot be parsed back.
5. Both run entirely offline on the client (Decision D5).

### 7.2 Text DSL Grammar

```
query        ::= SELECT fields FROM source WHERE? filters? ORDER_BY? orderings? LIMIT? number?

fields       ::= '*' | field_list
field_list   ::= field_expr (',' field_expr)*
field_expr   ::= field_id | field_id AS alias | aggregate '(' field_expr ')'

source       ::= '#' tag_id                          // all nodes with this supertag
               | node_id                             // single node's children
               | '(' query ')'                       // subquery

filters      ::= filter (('AND' | 'OR') filter)*
filter       ::= field_id op value
               | field_id 'IN' '(' value_list ')'
               | field_id 'BETWEEN' value 'AND' value   // range query → D3
               | field_id IS NULL
               | field_id IS NOT NULL
               | '(' filters ')'
               | NOT filter

op           ::= '=' | '!=' | '<' | '<=' | '>' | '>=' | 'CONTAINS' | 'STARTS_WITH'

orderings    ::= ordering (',' ordering)*
ordering     ::= field_id ('ASC' | 'DESC')?

value        ::= string_literal | number_literal | date_literal | bool_literal | node_ref

aggregate    ::= 'SUM' | 'AVG' | 'MIN' | 'MAX' | 'COUNT' | 'CONCAT'
```

**Example DSL queries**:

```sql
-- All tasks tagged #Task, priority High, due this week, sorted by due date
SELECT title, due_date, priority, assignee
FROM #Task
WHERE priority = "High" AND due_date BETWEEN 2026-04-06 AND 2026-04-12
ORDER BY due_date ASC

-- Count of completed tasks per assignee
SELECT assignee, COUNT(*) AS done_count
FROM #Task
WHERE status = "Done"
ORDER BY done_count DESC
LIMIT 10
```

### 7.3 Visual Query Builder

The visual query builder is a form-based UI component in the Presentation Layer. Its internal representation is a **QueryIR** (intermediate representation) that is the canonical form shared between the DSL parser and the visual builder:

```typescript
interface QueryIR {
  fields:    FieldSpec[];
  source:    SourceSpec;
  filters:   FilterGroup;    // tree of AND/OR/NOT
  orderings: OrderingSpec[];
  limit:     number | null;
}

interface FilterGroup {
  op:      'AND' | 'OR' | 'NOT';
  children: (FilterLeaf | FilterGroup)[];
}

interface FilterLeaf {
  field:    FieldId;
  operator: ComparisonOp;
  value:    QueryValue;
}
```

#### 7.3.1 Visual Builder Components

| Component | Responsibility |
|---|---|
| `SourcePicker` | Dropdown of all supertags; searches by name |
| `FieldSelector` | Checkbox list of available fields for selected source |
| `FilterBuilder` | Drag-and-drop AND/OR/NOT tree; per-leaf operator + value picker |
| `SortBuilder` | Ordered list of sort fields with ASC/DESC toggle |
| `LimitInput` | Optional numeric input |
| `DSLPreview` | Live-rendered DSL text; editable (triggers DSL→IR parse on change) |

#### 7.3.2 Round-Trip Protocol

```
User edits Visual Builder:
  visual_state → serialize → QueryIR → pretty_print → DSL text (DSLPreview updates)

User edits DSL text:
  DSL text → parse → QueryIR (or parse error shown inline) → visual_state (builder updates)
```

The `DSLPreview` component debounces text edits with a 200 ms delay before parsing to avoid thrashing during typing.

### 7.4 Query Execution Pipeline

```
Query text or QueryIR
  → Validate (field names exist, types match operators)
  → Optimize (push filters to indexes, choose range vs exact index)
  → Plan (sequence of graph API calls)
  → Execute (call GraphReadAPI methods, incl. query_field_range for BETWEEN)
  → Post-process (aggregation, ordering, limit)
  → ResultSet
```

**Index selection during optimization**:

| Filter predicate | Index used |
|---|---|
| `field = value` | `FieldValueIndex.exact` |
| `field IN (...)` | Multiple `exact` lookups, union |
| `field BETWEEN lo AND hi` | `RangeIndex` (Decision D3) |
| `field < value` | `RangeIndex`, unbounded lower |
| `field > value` | `RangeIndex`, unbounded upper |
| `tag = #T` | `TagIndex` |

---

## 8. Range Query Index

*(Decision D3: per-field BTree / sorted Vec)*

### 8.1 Index Structure

See §5.3.4 for the data structure. This section specifies the full maintenance and query contracts.

### 8.2 Indexed Field Types

Only fields of type `Number` and `Date` are indexed in the `RangeIndex`. Text fields use `FieldValueIndex.exact` for equality only; range predicates on text fields (e.g., `BETWEEN "a" AND "m"`) are executed as full-scan over the exact index's key space (less common; accepted trade-off).

### 8.3 Index Operations

| Operation | Complexity | Description |
|---|---|---|
| `insert(field, value, node)` | O(log n) | Insert node into BTreeMap at key value |
| `remove(field, value, node)` | O(log n) | Remove node from set at key value |
| `range_scan(field, lo, hi)` | O(log n + k) | Iterate BTreeMap range, union all node sets |
| `point_query(field, value)` | O(log n) | Equivalent to range with lo = hi |

### 8.4 Consistency Guarantees

The range index is updated **within the same mutation transaction** as the graph state change. There is no window where the graph state and the range index are inconsistent. On the client WASM build, this is enforced by the single-threaded event loop; on the server, by a per-shard mutex.

### 8.5 Snapshot and Replay

The range index is a derived structure; it is **not** persisted to the event log. On startup, it is rebuilt by replaying all `SetFieldValue` events for Number and Date fields. Incremental snapshots of the entire graph state (including the range index) are written to disk to bound replay time to a configurable maximum (default: 30 seconds of events).

---

## 9. Validation System

### 9.1 Validation Policies

Each field in a supertag definition carries one of three policies:

| Policy | Enforcement | Behaviour on violation |
|---|---|---|
| **Open** | Advisory | Warning shown in UI; write proceeds |
| **Guarded** | Soft-enforced | Error shown; write blocked in normal UI; API can override with explicit `force` flag |
| **Strict** | Hard-enforced | Error shown; write always blocked; DuckDB materialized view maintained |

### 9.2 Schema-on-Read / Schema-on-Write Boundary

- **Before supertag application**: any node can hold any fields with any values. No validation runs.
- **At supertag application time**: the Validation Layer checks all existing field values against the supertag's field definitions. Violations are reported. For Open fields, the tag is applied with warnings. For Guarded or Strict fields, the tag is applied only after the user resolves violations (or forces via API).
- **After supertag application**: mutations that set field values on tagged nodes are checked against the applied supertag's policies before the mutation is committed to the graph.

### 9.3 Validation Algorithm

```
validate_mutation(node, mutation):
  policies ← collect_field_policies(node.supertags)  // merged (see §10)
  for each (field_id, new_value) in mutation.field_changes:
    policy ← policies[field_id]
    violations ← check_type(field_id, new_value) + check_constraints(field_id, new_value)
    if violations is not empty:
      match policy:
        Open   → emit_warning(violations); allow
        Guarded → emit_error(violations); block unless mutation.force = true
        Strict  → emit_error(violations); always block
```

### 9.4 Referential Integrity (E_I Edges)

Nodes connected by E_I edges have enforced referential constraints. On `DeleteNode(target)`:

| Constraint mode | Behaviour |
|---|---|
| `Cascade` | Delete all source nodes referencing this target |
| `Restrict` | Block deletion if any source node references target |
| `Nullify` | Set referencing field to null on source nodes |

Constraint mode is set at E_I edge creation time and cannot be changed without deleting and re-creating the edge.

---

## 10. Multi-Supertag Merge and Precedence

*(Decision D2: prompt user; store per-field resolution)*

### 10.1 Problem Statement

A node may have multiple supertags applied simultaneously. Two supertags may define fields with the same `field_id` (or same human-readable name) but different types, policies, defaults, or constraints. The system must resolve these conflicts deterministically and persistently.

### 10.2 Merge Resolution Protocol

When `ApplySupertag(node_id, new_tag_id)` is called and the new tag introduces field definitions that conflict with existing applied tags, the following protocol runs:

```
merge_supertags(node, new_tag):
  conflicts ← find_field_conflicts(node.supertags, new_tag)

  if conflicts is empty:
    apply_tag(node, new_tag)       // no prompt needed
    return OK

  // D2: prompt user
  pending_merge ← PendingMerge {
    node_id:   node.id,
    new_tag:   new_tag.id,
    conflicts: conflicts,          // list of (field_id, existing_def, new_def)
  }
  emit_event(MergePromptRequired(pending_merge))
  return Err(MergePromptRequired)  // tag NOT applied yet

// User resolves via UI:
resolve_merge(node_id, resolutions: Vec<FieldResolution>):
  for each (field_id, chosen_def, chosen_policy) in resolutions:
    store_resolution(node_id, field_id, chosen_def, chosen_policy)
  apply_tag(node, pending_merge.new_tag)
  return OK
```

### 10.3 Per-Field Resolution Storage

Resolutions are stored as a dedicated node in the graph (making them queryable and replicable):

```
FieldResolutionNode {
  id:         NodeId,            // new node for this resolution record
  node_ref:   NodeId,            // the node whose fields were resolved
  field_id:   FieldId,
  chosen_tag: TagId,             // which supertag's definition was chosen
  overrides:  FieldDefinition,   // effective definition after resolution
  resolved_at: HLCTimestamp,
  resolved_by: AgentId,
}
```

These resolution nodes:
- Are replicated via CRDT like all other nodes.
- Are queried by the Validation Layer on every subsequent mutation to that node.
- Can be amended by the user at any time (amending triggers re-validation).

### 10.4 Conflict Detection Rules

Two field definitions conflict if they share the same `field_id` AND any of:
- Different `value_type`
- Different `required` setting
- Different `policy` (stricter wins as default suggestion, but user can override)
- Different `default` value

Fields from different tags that share only a human-readable name but have different `field_id` values do **not** conflict; they co-exist as separate fields (distinguished by tag-qualified name in the UI).

---

## 11. Sync and Conflict Resolution

*(Decision D4: conflict markers as node metadata + CRDT flag)*

### 11.1 CRDT Foundation

The Sync Layer uses **Yjs** CRDTs for conflict-free collaborative editing:

- Node content (rich text): Yjs `Y.Text` with character-level CRDT.
- Node presence/deletion: Yjs `Y.Map` keyed by `NodeId`.
- Edge presence/deletion: Yjs `Y.Map` keyed by `EdgeId`.
- Field values (scalar): Last-Write-Wins register using HLC timestamps as the ordering.
- Field resolution records (§10.3): LWW register per field.
- Conflict markers (§3.1): Yjs `Y.Map` with `is_conflicted` boolean — replicated as CRDT flag (Decision D4).

### 11.2 Merge Protocol

Sync events follow this protocol on receipt:

```
on_receive_sync_event(remote_event):
  1. Apply remote_event to local Yjs document → merged_state
  2. Detect structural conflicts:
       For each node N where merged_state differs from pre-merge local state:
         if validation_layer.check(N, merged_state) fails:
           attach_conflict_marker(N, conflicting_ops=[local_op, remote_event.op])
  3. Apply merged_state to Graph Runtime Layer
  4. Trigger incremental formula re-evaluation for dirty nodes
  5. Notify all affected query subscriptions
```

### 11.3 Conflict Markers (Decision D4)

Conflict markers are **first-class node metadata** (see §3.1). They are:

- **Stored in the graph** as `node.metadata.conflict_marker`.
- **Replicated via CRDT** as a boolean flag in `Y.Map`; the `conflicting_ops` list is replicated as a `Y.Array`.
- **Visible in all views** — the UI renders a conflict indicator on affected nodes.
- **Queryable** — `SELECT * FROM #AnyTag WHERE conflict_marker.is_conflicted = true` is a valid query.
- **Persistent** until explicitly resolved by a user or automation.

### 11.4 Conflict Resolution UI

When a conflict marker is present on a node, the UI presents a **Conflict Resolution Panel** showing:

1. The node's current merged state.
2. Each conflicting operation with its author, timestamp, and the value it set.
3. A "Choose" button per conflicting value, plus "Merge manually" and "Accept current" options.

On resolution:

```
resolve_conflict(node_id, resolution_strategy):
  clear_conflict_marker(node_id)
  apply_chosen_value(node_id, resolution_strategy.chosen_value)
  emit_event(ConflictResolved { node_id, strategy, agent_id, timestamp })
```

### 11.5 Validation-Aware Conflict Resolution

CRDT merge is always applied first (convergent). If the merged state violates a Strict-policy field:

1. Conflict marker is attached (D4).
2. The node's Strict-tagged field reverts to its last valid value (displayed as "current").
3. The conflicting merged value is stored in the `conflicting_ops` list for user review.
4. A Guarded-or-Open field would accept the merged value and show a warning instead.

---

## 12. Offline Parity Model

*(Decision D5: partial — validation + queries local; sandbox formulas server-only)*

### 12.1 Capability Matrix

| Feature | Online | Offline | Offline indicator |
|---|---|---|---|
| Read/browse any node | ✅ | ✅ | None |
| Create/edit nodes | ✅ | ✅ (queued) | "Pending sync" badge |
| Apply supertags | ✅ | ✅ | None |
| Field validation (Open/Guarded/Strict) | ✅ | ✅ | None |
| Tag index queries | ✅ | ✅ | None |
| Field exact-match queries | ✅ | ✅ | None |
| Range queries (BETWEEN, <, >) | ✅ | ✅ (D3 index local) | None |
| Visual query builder | ✅ | ✅ | None |
| Formula evaluation (Extism sandbox) | ✅ | ⚠️ Deferred | "Formula pending sync" |
| Conflict marker display | ✅ | ✅ | None |
| Conflict resolution | ✅ | ✅ (queued) | "Pending sync" badge |
| DuckDB SQL view | ✅ | ❌ | "Database view requires connection" |
| AI / LLM features | ✅ | ❌ | "AI requires connection" |
| Extism plugin execution | ✅ | ❌ | "Plugin requires connection" |
| Supertag merge prompt | ✅ | ✅ | None |
| Per-field resolution storage | ✅ | ✅ (queued) | None |

### 12.2 Offline Queue

Mutations created offline are appended to a local **OfflineQueue** (persisted in SQLite):

```rust
struct OfflineQueue {
    pending: VecDeque<QueuedMutation>,
}

struct QueuedMutation {
    mutation:   Mutation,
    created_at: HLCTimestamp,
    retry_count: u32,
}
```

On reconnection:

```
flush_offline_queue():
  sort pending by created_at (HLC order)
  for each queued_mutation in pending:
    result ← server.apply_mutation(queued_mutation.mutation)
    if result is conflict:
      attach_conflict_marker(affected_node, ...)
    else:
      remove from queue
```

### 12.3 WASM Graph Engine Guarantee

The client WASM build of `graph-engine-core` (§5.2) guarantees that the local SQLite event log can be replayed to reconstruct the full graph state, including all indexes, offline. This means the graph the user interacts with offline is semantically identical to the server graph (modulo pending mutations and stale formula caches).

---

## 13. Storage Layer

### 13.1 Storage Tiers

The system uses three storage tiers with strict derivation relationships:

```
Tier 0 (Source of Truth): Event Log
  ↓ replay
Tier 1 (Derived): In-Memory Graph Runtime (graph-engine-core)
  ↓ projection (Strict nodes only)
Tier 2 (Derived): DuckDB Relational Projection
```

No data flows upward. Tier 1 and Tier 2 can always be rebuilt by replaying Tier 0.

### 13.2 Event Log

| Environment | Backend | Notes |
|---|---|---|
| Server (cloud) | PostgreSQL | Append-only; partitioned by workspace; WAL archival |
| Client (local) | SQLite | Append-only; single file per workspace; WAL mode |

**Event schema (PostgreSQL)**:

```sql
CREATE TABLE events (
  event_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  workspace_id   UUID NOT NULL,
  agent_id       UUID NOT NULL,
  hlc_timestamp  BIGINT NOT NULL,    -- Hybrid Logical Clock, nanoseconds
  event_type     TEXT NOT NULL,
  payload        JSONB NOT NULL,
  sequence_num   BIGINT GENERATED ALWAYS AS IDENTITY
);

CREATE INDEX events_workspace_seq ON events (workspace_id, sequence_num);
CREATE INDEX events_hlc ON events (workspace_id, hlc_timestamp);
```

Events are **immutable** after insertion. Corrections are applied as new compensating events; the original event is never modified or deleted.

### 13.3 In-Memory Graph Runtime

The in-memory graph is rebuilt at startup by replaying events in `(workspace_id, sequence_num)` order. To bound startup time:

- **Snapshots** are written periodically (configurable; default: every 10,000 events or 5 minutes, whichever comes first).
- A snapshot is a binary serialization of the entire `NodeStore`, `EdgeStore`, `TagIndex`, `FieldValueIndex`, and `RangeIndex` state.
- On startup, the latest snapshot is loaded, then only events since the snapshot are replayed.

Snapshot format: MessagePack (compact, schema-agnostic, Rust native support).

### 13.4 DuckDB Relational Projection (Tier 2)

For every supertag whose policy is `Strict`, the system maintains a DuckDB table:

```sql
-- Auto-generated from supertag definition
CREATE TABLE task (
  node_id     TEXT PRIMARY KEY,
  title       TEXT NOT NULL,
  due_date    DATE,
  priority    TEXT CHECK (priority IN ('Low', 'Medium', 'High')),
  assignee    TEXT,
  status      TEXT NOT NULL DEFAULT 'Open',
  created_at  TIMESTAMP NOT NULL,
  updated_at  TIMESTAMP NOT NULL
);
```

DuckDB tables are updated synchronously with mutations on the server. They are **not available offline** (Decision D5).

Use cases for Tier 2:
- External BI tool integration (DuckDB can be queried over JDBC/ODBC).
- Complex analytical queries not expressible in the query DSL.
- Audit/export pipelines.

### 13.5 Snapshot and Recovery

Full recovery procedure:

```
1. Load latest snapshot (if any) into graph-engine-core
2. Replay events from event log since snapshot sequence_num
3. Rebuild DuckDB tables from in-memory graph (if DuckDB is out of sync)
4. Resume accepting mutations
```

Recovery time target: < 5 seconds for workspaces with ≤ 1,000,000 nodes (met by snapshot strategy in §13.3).

---

## 14. Frontend Architecture

### 14.1 Framework and Reactivity

The frontend is built with **SolidJS** using fine-grained reactivity. SolidJS signals map directly to graph node fields; signal updates are triggered by the Computation Layer's live query subscription callbacks.

```
graph subscription callback
  → update SolidJS signal (node field value)
  → SolidJS DOM patch (only affected DOM nodes updated)
```

This avoids Virtual DOM diffing overhead and allows table views with 100,000+ rows to update single cells without re-rendering the table.

### 14.2 Rich Text Editing

Node content is edited with **TipTap** (ProseMirror-based). TipTap extensions provide:

- Inline supertag application (`#tagname` syntax).
- Inline formula insertion (`=` prefix).
- Wiki-style node references (`[[node name]]`).
- Bidirectional sync with graph-engine-core via a custom TipTap plugin that maps ProseMirror transactions to graph mutations.

### 14.3 View Types

All view types render the same underlying graph data. Switching views does not change data — only the rendering policy.

| View | Primary use | Key components |
|---|---|---|
| **Outliner** | Hierarchical notes, free-form writing | Recursive E_H tree render, TipTap inline editing |
| **Table** | Structured data, database-like | SolidJS virtualized table, column = field, row = node |
| **Kanban** | Workflow, status-driven tasks | Swimlanes by enum field value, drag-to-set-value |
| **Calendar** | Date-driven tasks, scheduling | Monthly/weekly grid, Date field as position |
| **Chart** | Analytics, aggregation | Recharts adapter; data from query result sets |
| **Query Builder** | Constructing queries | Visual query builder UI (§7.3) |
| **Conflict Panel** | Resolving sync conflicts | Conflict resolution UI (§11.4) |

### 14.4 State Management

```typescript
// SolidJS store — mirrors the Graph Runtime Layer state
const [graphStore, setGraphStore] = createStore<GraphState>({
  nodes:      new Map<NodeId, NodeData>(),
  selections: new Set<NodeId>(),
  activeView: ViewType.Outliner,
  pendingSync: new Set<NodeId>(),    // D5: offline queue visibility
  conflicts:  new Set<NodeId>(),     // D4: conflict marker visibility
});
```

All mutations go through a single `dispatch(mutation: Mutation)` function that:
1. Optimistically updates the local `graphStore`.
2. Sends the mutation to the graph engine (client WASM).
3. Enqueues the mutation for server sync if offline, or sends immediately if online.
4. On server acknowledgement, confirms the optimistic update or rolls back on conflict.

### 14.5 Offline Indicator

A persistent banner is shown when the client is offline:

```
⚠️  You are offline. Changes are saved locally and will sync when reconnected.
    Formulas show cached values. [N formulas pending sync]
```

Formula nodes with stale cached values are rendered with a distinguishing style (e.g., italic + clock icon).

---

## 15. AI and Automation Layer

### 15.1 Cross-Cutting Scope

The AI/Automation Layer cuts across all other layers. It can:

- Read from any layer (via `GraphReadAPI`).
- Write via the standard mutation pipeline (with `agent_id` set to the automation's identity).
- Register query subscriptions to trigger automations reactively.
- Invoke formula evaluation (server-only, D5).

### 15.2 LLM Integration

LLM access is mediated by **LiteLLM**, which provides:

- Provider-agnostic API (OpenAI, Anthropic, Mistral, local Ollama — same interface).
- Credential management without exposing keys to the frontend.
- Token usage tracking per workspace.

**Model Context Protocol (MCP)** is used to give LLMs structured access to the graph:

```
LLM ←→ MCP Server ←→ GraphReadAPI / GraphMutateAPI
```

MCP tools exposed to LLMs:

| Tool | Description |
|---|---|
| `search_nodes` | Query DSL execution; returns ResultSet |
| `get_node` | Fetch full node with all fields |
| `create_node` | Create node with optional supertag |
| `set_field` | Set a field value on a node |
| `summarize_subgraph` | Render a subtree as markdown for context |

### 15.3 Plugin Sandbox (Extism)

Custom automation logic is packaged as **Extism** WASM plugins. Plugins:

- Run only on the server (Decision D5).
- Have explicit capability grants: read-only graph access, write-limited graph access, network access (allowlist), file system access (sandboxed directory).
- Are subject to resource limits: CPU time (configurable; default 5 s), memory (configurable; default 64 MB).
- Are loaded from a plugin registry (hosted or self-hosted).

### 15.4 Automation Triggers

Automations are triggered by:

| Trigger type | Description |
|---|---|
| `OnMutation(filter)` | Run when a mutation matching a filter is applied |
| `OnQueryResult(query, condition)` | Run when a live query result changes and meets a condition |
| `OnSchedule(cron)` | Run on a cron schedule |
| `OnConflict(tag_filter)` | Run when a conflict marker is attached to a matching node |
| `OnFormulaStaleness` | Run when formula nodes remain stale beyond a threshold |

---

## 16. API Reference

### 16.1 REST / WebSocket API

The server exposes a REST+WebSocket API. WebSocket is used for:
- Real-time sync of Yjs CRDT updates.
- Live query subscription result delivery.
- Formula evaluation result delivery.

REST is used for:
- Authentication and workspace management.
- Bulk import/export.
- DuckDB query execution.
- Plugin registry management.

### 16.2 Core Endpoints

```
POST   /api/v1/workspaces/{ws}/mutations          Apply one or more mutations
GET    /api/v1/workspaces/{ws}/nodes/{id}         Get a node
POST   /api/v1/workspaces/{ws}/queries            Execute a query (DSL text body)
WS     /api/v1/workspaces/{ws}/sync               Yjs CRDT sync channel
WS     /api/v1/workspaces/{ws}/subscriptions      Live query subscription channel
POST   /api/v1/workspaces/{ws}/formulas/evaluate  Server-side formula evaluation
POST   /api/v1/workspaces/{ws}/plugins/{id}/invoke Invoke Extism plugin
GET    /api/v1/workspaces/{ws}/conflicts          List all conflicted nodes
POST   /api/v1/workspaces/{ws}/conflicts/{id}/resolve  Resolve a conflict
```

### 16.3 Mutation Payload

```json
{
  "mutations": [
    {
      "type": "SetFieldValue",
      "node_id": "018e3f2a-1234-7abc-def0-111122223333",
      "field_id": "due_date",
      "value": "2026-04-30",
      "force": false
    }
  ],
  "idempotency_key": "client-generated-uuid"
}
```

The `idempotency_key` allows safe retries; the server will return the same result for duplicate requests within a 24-hour window.

### 16.4 Query Request and Response

```json
// Request
{
  "query": "SELECT title, due_date FROM #Task WHERE priority = \"High\" ORDER BY due_date ASC LIMIT 20",
  "subscribe": true    // if true, returns subscription_id for WebSocket updates
}

// Response
{
  "results": [
    { "node_id": "...", "title": "Ship v1", "due_date": "2026-04-15" },
    ...
  ],
  "subscription_id": "sub-uuid-if-subscribe-true",
  "total_count": 47,
  "page_info": { "has_more": true, "next_cursor": "..." }
}
```

---

## 17. Technology Stack

### 17.1 Full Stack Reference

| Layer | Technology | Role |
|---|---|---|
| **Frontend framework** | SolidJS | Fine-grained reactive UI |
| **Rich text editor** | TipTap (ProseMirror) | Node content editing |
| **Graph engine** | Rust → WASM (`graph-engine-core`) | Client-side graph, validation, queries, range index |
| **Graph engine (server)** | Rust native (`graph-engine-core` + `server` feature) | Server-side graph, authoritative mutations |
| **CRDT sync** | Yjs | Conflict-free collaborative editing |
| **Event log (cloud)** | PostgreSQL | Immutable append-only event log, server |
| **Event log (local)** | SQLite (WAL mode) | Immutable append-only event log, client |
| **Relational projection** | DuckDB | Materialized views for Strict-tagged nodes, analytics |
| **Plugin sandbox** | Extism | WASM-based automation plugins (server-only) |
| **LLM gateway** | LiteLLM | Provider-agnostic LLM API |
| **LLM context protocol** | MCP + REST + WebSocket | Structured graph access for AI agents |
| **Desktop wrapper** | Tauri | Cross-platform desktop app shell (uses local SQLite) |
| **Container runtime** | Docker | Server deployment |

### 17.2 Rust Crate Structure

```
graph-engine-core/         # Core Rust library (no_std compatible subset)
  src/
    lib.rs                 # Feature-gated exports
    node_store.rs
    edge_store.rs
    tag_index.rs
    field_index.rs
    range_index.rs         # D3: BTreeMap-based range index
    dependency_dag.rs
    mutation.rs
    validation.rs
    supertag_merge.rs      # D2: merge prompt + resolution storage
    conflict_marker.rs     # D4: conflict marker node metadata
    offline_queue.rs       # D5: offline mutation queue
    query/
      dsl_parser.rs        # D1: text DSL parser → QueryIR
      query_ir.rs
      query_executor.rs
      query_optimizer.rs
  Cargo.toml               # [features]: server, client

server/                    # Rust binary — HTTP + WebSocket server
  src/
    main.rs
    sync_handler.rs
    formula_evaluator.rs
    plugin_runner.rs
    persistence/
      postgres.rs
      duckdb.rs

client/                    # TypeScript + WASM glue
  src/
    graph_engine.ts        # wasm-bindgen generated bindings
    query_builder/
      QueryBuilder.tsx     # D1: visual builder component
      DSLPreview.tsx
      FilterBuilder.tsx
    views/
      OutlinerView.tsx
      TableView.tsx
      KanbanView.tsx
      CalendarView.tsx
      ChartView.tsx
    conflict/
      ConflictPanel.tsx    # D4: conflict resolution UI
    offline/
      OfflineBanner.tsx    # D5: offline indicator
```

---

## 18. Security and Multi-Tenancy

### 18.1 Workspace Isolation

Every node, edge, event, and query belongs to a **workspace**. Workspaces are the unit of isolation:

- All event log queries are scoped by `workspace_id`.
- The in-memory graph is per-workspace (separate `graph-engine-core` instance per workspace on the server).
- DuckDB databases are per-workspace.
- Yjs documents are per-workspace.

Cross-workspace references (E_R edges pointing to a different workspace) are not supported in v1.

### 18.2 Authentication and Authorization

| Mechanism | Usage |
|---|---|
| JWT (RS256) | API authentication; issued by auth service |
| RBAC per workspace | `Owner`, `Editor`, `Commenter`, `Viewer` roles |
| Plugin capability grants | Explicit per-plugin, per-workspace grants |
| LLM API key management | Server-side only; never exposed to frontend |

### 18.3 Mutation Authorization

Every mutation carries an `agent_id`. The server checks:

```
authorize_mutation(agent_id, workspace_id, mutation):
  role ← get_role(agent_id, workspace_id)
  match mutation:
    CreateNode | SetFieldValue | AddEdge → require role ≥ Editor
    DeleteNode | RemoveEdge             → require role ≥ Editor
    ApplySupertag                       → require role ≥ Editor
    ResolveConflict                     → require role ≥ Editor
    DeleteWorkspace                     → require role = Owner
```

### 18.4 Extism Plugin Security

Plugins are isolated via WASM memory safety plus explicit capability grants. No plugin can:

- Access another workspace's data.
- Make network requests to non-allowlisted hosts.
- Execute beyond its CPU/memory limits.
- Persist state outside the graph mutation API.

Plugin code is hashed on upload and verified on every load.

---

## 19. Deployment and Operations

### 19.1 Server Deployment

```
Docker Compose (development):
  - server (Rust binary)
  - postgres
  - redis (session cache)
  - minio (file attachments)

Kubernetes (production):
  - server Deployment (auto-scaled by workspace count)
  - PostgreSQL (managed, e.g., RDS or Cloud SQL)
  - Redis Cluster (session + pub/sub)
  - Object storage (S3-compatible for snapshots and file refs)
```

### 19.2 Desktop (Tauri)

The Tauri desktop app bundles:
- The client WASM build of `graph-engine-core`.
- SQLite for local event log.
- The SolidJS frontend.
- A local server process (Rust, `server` feature disabled; WASM engine only).

In desktop mode, the user can work fully offline (§12). Sync happens when a network connection is available.

### 19.3 Observability

| Signal | Technology | Key metrics |
|---|---|---|
| Metrics | Prometheus | Mutation latency, query latency, formula eval time, CRDT sync lag |
| Tracing | OpenTelemetry (Jaeger) | Per-mutation trace spanning all layers |
| Logging | Structured JSON (tracing-rs) | Per-event log with workspace_id, agent_id, event_type |
| Alerting | Grafana | Alert on p99 mutation latency > 100 ms, sync lag > 5 s |

### 19.4 Performance Targets

| Operation | Target p50 | Target p99 |
|---|---|---|
| Node lookup (graph engine) | < 0.1 ms | < 1 ms |
| Tag index query | < 1 ms | < 5 ms |
| Range query (RangeIndex) | < 2 ms | < 10 ms |
| Mutation round-trip (online) | < 50 ms | < 200 ms |
| Formula evaluation (server) | < 500 ms | < 5 s |
| Graph rebuild from snapshot | < 1 s | < 5 s |
| Cold start (no snapshot) | < 30 s | < 2 min |

### 19.5 Data Retention and Backup

- **Event log**: retained indefinitely (append-only; deletions are soft via compensating events).
- **Snapshots**: retained for 30 days; oldest pruned automatically.
- **PostgreSQL backups**: daily full backup + continuous WAL archival (point-in-time recovery to any second within 30 days).
- **DuckDB**: rebuilt on demand; not independently backed up.

---

*End of Full System Architecture — all decisions applied, no TBDs remaining.*
