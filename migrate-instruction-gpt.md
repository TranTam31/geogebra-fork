# GeoGebra Modern Migration Instruction

This document is the primary migration blueprint for rebuilding GeoGebra on a modern, maintainable stack while preserving mathematical behavior, app-variant identity, and cross-platform reliability.

The migration is not a big-bang rewrite. It is a controlled replacement of architecture layers with strict compatibility and quality gates.

## 1. Mission

Build a new system around a strict engine boundary, then rebuild all user-facing products as thin adapters around that engine.

Target direction:
- Core engine and kernel: Rust
- Browser shell: TypeScript + React
- Engine runtime in browser: WebAssembly
- Rendering: platform-neutral scene model with web/desktop backends
- Desktop and mobile: separate shells using the same engine API
- Shared contracts: versioned schemas and API boundaries

## 2. Current Repository Reality

The current repository is a Gradle composite build with three main trees:
- `source/shared`
- `source/web`
- `source/desktop`

Important facts that shape migration:
- Web is GWT-based Java to JavaScript, not a Node/React frontend.
- Desktop is JVM-based with native JOGL and Giac integration.
- Shared code includes both mathematical behavior and presentation-related concerns.
- Legacy documents rely on XML + ZIP structures and compatibility parsing behavior.

## 3. Architecture Laws

These are mandatory.

1. UI layers must never call engine internals directly.
2. Semantic state and presentation state must be separated by schema.
3. Every external contract must be versioned.
4. Every feature migration must be test-first.
5. Determinism is required for engine operations.
6. Compatibility is required until explicit parity gates pass.
7. Variant behavior must be data-driven, not hardcoded in multiple places.
8. CAS unavailability must produce explicit, safe fallback behavior.
9. Legacy removal happens only after measurable parity sign-off.

## 4. Scope Model

### 4.1 Initial Release Scope
- Engine foundation for core algebra/geometry/document operations
- WASM runtime integration for browser
- TypeScript + React web shell
- Minimal but real variant coverage: Graphing, Geometry, Scientific baseline
- Legacy import and round-trip for supported feature subset

Out of initial scope:
- Full desktop parity
- Full mobile parity
- Full 3D parity
- Full CAS parity if it blocks core schedule
- XR

### 4.2 Expansion Scope
- Full browser variant set
- Desktop shell rollout on shared engine API
- Broader CAS parity
- Expanded legacy compatibility coverage

### 4.3 Platform Completion Scope
- Native mobile adapters on shared contracts
- Advanced/experimental platform surfaces

## 5. Contracts That Must Be Frozen Before Engine Porting

### 5.1 Engine API Contract
Define and freeze:
- command execution interface
- object lifecycle and identity model
- dependency graph update protocol
- undo/redo contract
- event and notification model
- error model and codes

### 5.2 Document Contract
Define and freeze:
- canonical object schemas
- serialization invariants
- schema versioning strategy
- migration rules between schema versions

### 5.3 Presentation Contract
Define and freeze:
- scene graph or draw-command schema
- style state model (color, visibility, layer, labels, thickness)
- hit-testing and interaction semantics

### 5.4 Editor and Keyboard Contracts
Define and freeze:
- editor AST and edit-command protocol
- keyboard layout data schema
- platform-agnostic input action protocol

### 5.5 Compatibility Contract
Define and freeze:
- legacy feature support matrix
- web embedding parameter support policy
- exported browser API compatibility table
- deprecation and removal policy

## 6. Mandatory State-Separation Phase

Objective:
- separate semantic mathematical state from visual/presentation state without behavior loss.

Deliverables:
- `core-state` schema
- `view-state` schema
- deterministic mapping from legacy XML into split-state model
- schema migration tooling
- mapping corpus tests from real legacy files

Exit criteria:
- legacy documents load into split state deterministically
- split state can re-serialize for supported features without semantic loss

## 7. Migration Phases

### Phase 0: Audit and Behavior Freeze
Objective:
- capture existing product behavior before replacement

Deliverables:
- feature inventory
- app variant inventory
- representative document corpus
- golden algebra/CAS outputs
- golden screenshots for key constructions

Exit criteria:
- scope labels assigned: keep, defer, drop

### Phase 1: Contract Freeze
Objective:
- finalize contracts in section 5

Exit criteria:
- all teams can build against frozen interfaces

### Phase 2: State Separation and Schema Bridge
Objective:
- complete section 6 deliverables

Exit criteria:
- split-state compatibility path validated

### Phase 3: Engine Foundation
Objective:
- implement deterministic Rust core for in-scope features

Deliverables:
- parser, kernel, document, serialization crates
- dependency and undo/redo logic
- contract-level integration adapters

Validation:
- unit tests, differential tests, fuzz tests

Exit criteria:
- engine workflows pass without UI runtime

### Phase 4: Browser Shell
Objective:
- deliver first modern user-facing product

Deliverables:
- React app shell
- routing and variant bootstrap
- WASM engine bridge
- document lifecycle integration

Exit criteria:
- open/edit/save works for in-scope variants

### Phase 5: Renderer and Views
Objective:
- recover high-value visual behavior and interaction

Deliverables:
- renderer backend implementation
- core views and interaction behavior

Exit criteria:
- parity thresholds pass for in-scope constructions and interactions

### Phase 6: Editor and Keyboard
Objective:
- replace input stack with shared protocols

Exit criteria:
- editing, keyboard, and accessibility acceptance tests pass

### Phase 7: Variant Expansion
Objective:
- roll out remaining browser variants using manifest-driven configuration

Exit criteria:
- variant smoke and acceptance suites pass

### Phase 8: Desktop and Mobile Adapters
Objective:
- add platform shells around shared engine contracts

Exit criteria:
- cross-platform file consistency and key behavior consistency validated

### Phase 9: Legacy Import Hardening
Objective:
- expand import fidelity and unsupported feature diagnostics

### Phase 10: Legacy Removal
Objective:
- remove old runtime paths after parity and stability sign-off

## 8. Side-by-Side Coexistence Strategy

Use a strangler rollout.

Required approach:
1. Keep legacy stack runnable throughout migration.
2. Introduce new runtime behind explicit gating.
3. Route selected variants to new runtime first.
4. Keep telemetry, error reporting, and diagnostics unified.
5. Expand rollout only after quality gates pass.

## 9. CAS Strategy

CAS is high-risk and must be staged.

Step 1:
- implement essential CAS subset needed by in-scope features
- define explicit behavior for unsupported commands

Step 2:
- expand parity set based on usage data and test corpus

Step 3:
- select long-term architecture (native, embedded, hybrid) using measured reliability/performance results

## 10. Compatibility Requirements

### 10.1 File Compatibility
- preserve legacy import path
- support round-trip for in-scope feature set
- publish schema version notes and migration guarantees

### 10.2 Web Embedding Compatibility
- publish support table for important embedding parameters
- provide temporary shims for high-usage paths

### 10.3 Exported Browser API Compatibility
- publish method-level compatibility table (supported, changed, deprecated)
- provide adapter layer for high-impact behavior changes

### 10.4 Variant Compatibility
- preserve variant identity and startup defaults
- preserve key workflows per variant

## 11. Testing Strategy and Quality Gates

### 11.1 Mandatory Test Types
- engine unit tests
- parser fuzz and robustness tests
- schema migration tests
- differential tests versus legacy outputs
- screenshot regression tests
- browser end-to-end tests

### 11.2 Suggested Numeric Gates
- algebra command parity for in-scope set: >= 99%
- document round-trip parity for in-scope corpus: >= 98%
- critical browser E2E flows: 100% pass
- pre-release crash-free startup: >= 99.9%

### 11.3 Release Stop Conditions
Do not ship if any are true:
- parity below threshold
- unresolved data-loss defects
- unresolved deterministic mismatches in critical workflows
- unresolved migration blocker in top-priority variants

## 12. Recommended New Repository Layout

- `crates/engine`
- `crates/document`
- `crates/parser`
- `crates/cas`
- `crates/render-core`
- `crates/ffi`
- `apps/web`
- `apps/desktop`
- `apps/mobile-ios`
- `apps/mobile-android`
- `schemas/`
- `fixtures/`
- `tests/golden`
- `tests/e2e`
- `tooling/`
- `docs/`

If risk must be reduced, start web-first and defer desktop/mobile app shells until browser parity is stable.

## 13. First 120 Days Plan

Days 1-30:
- complete behavior audit and contract freeze
- build migration corpus and golden baselines

Days 31-60:
- complete state-separation schema bridge prototype
- validate deterministic mapping from legacy documents

Days 61-90:
- deliver first engine milestone for in-scope features
- integrate WASM runtime prototype

Days 91-120:
- deliver first end-to-end browser workflow (open/edit/save)
- run parity gates on initial corpus

## 14. Governance Checkpoints

Mandatory checkpoints:
- contract freeze
- state-separation bridge completion
- first engine milestone
- first browser usability milestone
- first parity gate review

At each checkpoint, capture:
- pass/fail outcomes
- risks and owners
- scope changes
- next checkpoint criteria

## 15. Practical Definition of Done

Migration is complete only when all are true:
- in-scope documents produce equivalent mathematical behavior
- supported variants pass acceptance suites
- engine is validated independently from UI runtimes
- compatibility commitments are met and documented
- platform adapters remain thin and replaceable
- legacy stack is removed only after final parity sign-off

## 16. Execution Order

Execute in this order:
1. behavior freeze and corpus capture
2. contract freeze
3. state separation and schema bridge
4. deterministic engine foundation
5. browser shell rollout
6. renderer/editor/keyboard completion
7. variant expansion
8. desktop/mobile adapters
9. legacy removal after hard parity gates

This sequence keeps risk controlled and turns the rewrite into measurable milestones instead of a single high-risk conversion.