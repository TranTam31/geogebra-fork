# GeoGebra Modern Migration Final Instruction

Status: Final merged blueprint (architecture + execution)

Audience: engineering leadership, architecture owners, platform teams, QA, release management

This document combines the strongest parts of the three migration drafts into one practical and enforceable plan:
- Strategic architecture laws and compatibility discipline
- Concrete phased execution with owners, timelines, and go/no-go gates
- Repository-grounded module mapping for the current GeoGebra codebase

---

## 1. Mission

Migrate GeoGebra to a modern, maintainable stack without losing mathematical behavior, variant identity, or cross-platform reliability.

This is **not** a big-bang rewrite. It is a controlled strangler migration where legacy and new runtimes coexist until parity gates are passed.

Primary target direction:
- Engine and kernel: Rust
- Browser shell: TypeScript + React
- Browser runtime: WebAssembly bridge to engine
- Rendering: platform-neutral scene contract with web/desktop backends
- Desktop/mobile: thin adapters on the same engine contracts
- Contracts and schemas: versioned and explicitly governed

---

## 2. Current Repository Reality (Non-Negotiable Baseline)

The current repository is a Gradle composite build:
- `source/shared`
- `source/web`
- `source/desktop`

Key facts that must shape migration planning:
- Web is GWT Java-to-JS, not a Node/React frontend today.
- Desktop is JVM-based with native JOGL and Giac integration.
- Shared modules mix semantic math state and visual/presentation concerns.
- Legacy documents are XML+ZIP based and deeply embedded in behavior.
- Public web embedding parameters and browser API surface are large and compatibility-sensitive.
- Existing variant model is already explicit and should be reused as migration routing metadata.

Implication:
- Web-first migration is mandatory for risk control.
- Desktop and mobile must stay out of the critical path until browser parity is stable.

---

## 3. Architecture Laws

These laws are mandatory for all teams and phases.

1. UI layers must not call engine internals directly.
2. Semantic state and presentation state must be separated by contract.
3. Every external contract must be versioned.
4. Feature migration is test-first; no behavior port without golden/differential tests.
5. Engine operations must be deterministic.
6. Compatibility commitments remain active until explicit parity sign-off.
7. Variant behavior must be manifest-driven, not duplicated in code branches.
8. CAS unavailability must produce explicit, safe fallback behavior.
9. Legacy runtime removal is allowed only after measurable parity gates pass.
10. Any contract-breaking change requires architecture review plus migration notes.

---

## 4. Scope Model

### 4.1 Wave 1 (Initial Shippable Scope)

- Engine foundation for core algebra/geometry/document operations
- WASM runtime integration for browser
- TypeScript + React shell for selected variants
- In-scope variants for first release:
  - Graphing
  - Geometry
  - Scientific
- Legacy import + round-trip for supported feature subset

Out of Wave 1:
- Full desktop parity
- Full mobile parity
- Full 3D parity
- Full CAS parity if it threatens schedule
- XR

### 4.2 Wave 2 (Expansion)

- Full browser variant coverage
- Desktop shell rollout over shared engine API
- Broader CAS parity and import fidelity
- Expanded embedding/API compatibility matrix

### 4.3 Wave 3 (Platform Completion)

- Native mobile adapters on shared contracts
- Advanced and experimental surfaces (including XR if still required)

---

## 5. Contracts That Must Be Frozen Before Engine Porting

No Rust engine port starts until these contracts are approved.

### 5.1 Engine API Contract

Freeze:
- Command execution interface
- Object lifecycle and identity model
- Dependency graph update protocol
- Undo/redo transaction contract
- Event/notification model
- Error model and codes

### 5.2 Document Contract (Dual-Format Strategy)

Freeze:
- Canonical internal document schema (versioned)
- Serialization invariants
- Schema migration rules
- Deterministic import mapping from legacy XML+ZIP

Important policy:
- Do not remove legacy XML+ZIP support in early phases.
- Use a dual-format bridge during migration:
  - Legacy format for compatibility ingestion and round-trip obligations
  - Canonical schema for internal modernization and new tooling

### 5.3 Presentation Contract

Freeze:
- Scene graph or draw-command schema
- Style model (visibility, color, label mode, layer, line style, opacity)
- Hit-testing and interaction semantics

### 5.4 Editor and Keyboard Contracts

Freeze:
- Editor AST and edit-command protocol
- Keyboard layout schema and action mapping
- Platform-agnostic input action protocol

### 5.5 Compatibility Contract

Freeze:
- Legacy feature support matrix
- Web embedding parameter support table
- Exported browser API compatibility matrix (method-level)
- Deprecation/removal policy

---

## 6. Mandatory State-Separation Program

Objective:
- Separate mathematical semantics from view/presentation state without behavior loss.

Required deliverables:
- `core-state` schema
- `view-state` schema
- Deterministic legacy XML -> split-state mapping
- Schema migration tooling
- Corpus tests from real documents

Exit criteria:
- Legacy files load deterministically into split-state model
- Supported features can round-trip without semantic loss
- Differential checks against legacy outputs pass parity gate

---

## 7. Module Mapping from Current Repo to Future Architecture

Use this mapping as planning truth for decomposition and staffing.

### 7.1 Shared Layer

- `source/shared/common`
  - Current: math kernel + object model + significant UI/state coupling
  - Target: reference behavior for Rust engine + compatibility fixtures

- `source/shared/common-jre`
  - Current: JVM helpers and broad test surface
  - Target: compatibility harness and differential-test adapters (temporary)

- `source/shared/editor-base`
  - Current: editor abstractions
  - Target: canonical grammar/AST/edit-command contract

- `source/shared/renderer-base`
  - Current: shared rendering primitives
  - Target: rendering contract (not backend implementation)

- `source/shared/canvas-base`
  - Current: common drawing abstraction
  - Target: low-level draw command contract

- `source/shared/keyboard-base`, `source/shared/keyboard-scientific`
  - Current: keyboard model and scientific extensions
  - Target: schema-driven keyboard definitions and action protocol

- `source/shared/giac-jni`
  - Current: JNI CAS bridge
  - Target: isolated CAS service boundary behind engine contract

### 7.2 Web Layer

- `source/web/web`
  - Current: GWT orchestration, multi-app bootstrapping, HTML generation
  - Target: functional specification for modern web shell behavior

- `source/web/web-common`
  - Current: embedding parameters and exported JS API helpers
  - Target: migration compatibility adapter and contract truth source

- `source/web/editor-web`, `source/web/keyboard-web`, `source/web/renderer-web`, `source/web/canvas-web`
  - Current: web implementations tied to legacy stack
  - Target: reference behavior for React-based adapters and renderer backend

- `source/web/uitest`
  - Current: Cypress/UI harness
  - Target: parity and regression gates for coexistence period

### 7.3 Desktop Layer

- `source/desktop/desktop`, `source/desktop/jogl2`, desktop renderer/editor modules
  - Current: JVM + native dependencies
  - Target: deferred adapter wave after browser parity stability

---

## 8. Execution Plan (Phased, Gated)

## Phase 0: Audit and Behavior Freeze (4-6 weeks)

Objective:
- Capture current behavior before replacement.

Deliverables:
- Feature and variant inventory
- Representative document corpus
- Golden algebra/CAS outputs
- Golden screenshots for high-value constructions
- Baseline performance metrics

Exit criteria:
- Scope labels assigned: keep/defer/drop
- Corpus and golden baselines versioned in repo

Go/No-Go:
- No Phase 1 start without approved inventory and corpus coverage.

## Phase 1: Contract Freeze (2-4 weeks)

Objective:
- Finalize all contracts in Section 5.

Deliverables:
- Engine API specification
- Document contract and migration policy
- Presentation contract
- Editor/keyboard contract
- Compatibility contract tables

Exit criteria:
- Teams can build stubs against frozen interfaces.

Go/No-Go:
- No Phase 2 start with unresolved contract ownership.

## Phase 2: State Separation and Schema Bridge (8-12 weeks)

Objective:
- Complete split-state bridge with deterministic mapping.

Deliverables:
- Core/view schemas
- Legacy mapping pipeline
- Migration tooling
- Round-trip tests for supported subset

Exit criteria:
- Deterministic load and round-trip parity for in-scope corpus.

Go/No-Go:
- No engine expansion before deterministic bridge is proven.

## Phase 3: Engine Foundation in Rust (4-8 months)

Sub-phases:
1. Object model and serialization
2. Expression parser and evaluator
3. Dependency graph + undo/redo
4. Legacy compatibility adapters
5. CAS integration (staged)
6. Render command export

Validation:
- Unit tests
- Differential tests vs legacy runtime
- Fuzz/robustness tests for parser and import

Exit criteria:
- In-scope engine workflows run headless and pass parity thresholds.

Go/No-Go:
- No browser rollout without passing headless parity gate.

## Phase 4: Browser Shell (3-5 months)

Sub-phases:
1. TS/React project + WASM bridge
2. Variant bootstrap/routing
3. Document lifecycle integration
4. Initial rendering and interaction
5. Input/editor/keyboard integration

Exit criteria:
- Open/edit/save fully functional for in-scope variants.

Go/No-Go:
- No broad rollout while critical workflows fail E2E parity.

## Phase 5: Renderer and Interaction Recovery (2-4 months)

Objective:
- Recover high-value visual and interaction behavior.

Exit criteria:
- Screenshot and interaction parity gates pass for in-scope constructions.

## Phase 6: Variant Expansion (ongoing)

Objective:
- Roll out remaining browser variants using manifest-driven configuration.

Exit criteria:
- Variant smoke and acceptance suites pass per variant gate.

## Phase 7: Desktop Adapter Wave (deferred)

Objective:
- Introduce desktop adapter on shared engine contracts.

Exit criteria:
- Cross-platform file consistency and key behavior consistency pass.

## Phase 8: Mobile Adapter Wave (deferred)

Objective:
- Introduce native mobile adapters on shared contracts.

Exit criteria:
- Mobile workflow acceptance and compatibility gates pass.

## Phase 9: Import Hardening and Compatibility Closure (ongoing)

Objective:
- Expand unsupported-feature diagnostics and import fidelity.

## Phase 10: Legacy Removal

Objective:
- Remove old runtime paths only after final parity sign-off.

Hard preconditions:
- Compatibility commitments met
- Stability targets met
- Rollback strategy validated

---

## 9. Side-by-Side Coexistence Strategy (Strangler)

Required operating model:
1. Keep legacy stack runnable throughout migration.
2. Introduce modern runtime behind explicit feature flags.
3. Route selected variants to new runtime first.
4. Keep telemetry, diagnostics, and error taxonomy unified.
5. Expand rollout only when quality gates pass.
6. Maintain immediate rollback path per variant.

Rollout order recommendation:
- Graphing -> Geometry -> Scientific -> Suite sub-app increments -> CAS -> remaining variants

---

## 10. CAS Strategy (Staged Risk Control)

Step 1:
- Implement essential CAS subset required for Wave 1 workflows.
- Define explicit fallback for unsupported commands.

Step 2:
- Expand parity based on usage data and golden corpus failures.

Step 3:
- Decide long-term architecture (native/embedded/hybrid) from reliability and performance evidence.

Rule:
- CAS parity work cannot destabilize core algebra/geometry schedule.

---

## 11. Compatibility Requirements

### 11.1 File Compatibility

- Preserve legacy import path during coexistence
- Support round-trip for in-scope feature set
- Publish schema/version migration guarantees

### 11.2 Web Embedding Compatibility

- Publish parameter-level support matrix
- Add adapter shims for high-usage embedding paths

### 11.3 Exported Browser API Compatibility

- Publish method-level matrix: supported/changed/deprecated
- Provide adapter layer for high-impact behavior changes

### 11.4 Variant Compatibility

- Preserve variant identity and startup defaults
- Preserve key workflows and expected initial perspectives

---

## 12. Testing Strategy and Numeric Gates

### 12.1 Mandatory Test Types

- Engine unit tests
- Parser/import fuzz tests
- Schema migration tests
- Differential tests versus legacy outputs
- Screenshot regression tests
- Browser end-to-end tests

### 12.2 Suggested Release Gates (Wave 1)

- Algebra command parity (in-scope set): >= 99%
- Document round-trip parity (in-scope corpus): >= 98%
- Critical browser E2E flows: 100% pass
- Pre-release crash-free startup: >= 99.9%

### 12.3 Stop-Ship Conditions

Do not ship if any of the following is true:
- Parity below threshold
- Unresolved data-loss defects
- Unresolved deterministic mismatches in critical workflows
- Unresolved migration blocker in top-priority variants

---

## 13. Team Model and Governance

### 13.1 Recommended Team Topology

- Architecture and contracts: 1-2 senior architects
- Engine core (Rust): 4-6 engineers
- Web shell and interaction: 3-5 engineers
- Compatibility and migration tooling: 2-3 engineers
- QA and parity automation: 2-3 engineers
- Release and observability support: shared platform role(s)

### 13.2 Governance Checkpoints

Mandatory checkpoints:
1. Contract freeze
2. State-separation bridge completion
3. First engine milestone
4. First browser usability milestone
5. First parity gate review

For each checkpoint capture:
- Pass/fail result
- Risks and owners
- Scope changes
- Entry criteria for next checkpoint

### 13.3 Go/No-Go Discipline

Each major phase must end with a formal decision:
- Continue
- Continue with narrowed scope
- Stop and remediate

No implicit progression.

---

## 14. Recommended Future Repository Layout

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

Practical migration note:
- Begin web-first; add desktop/mobile app shells only after browser parity stabilizes.

---

## 15. First 120 Days (Execution Snapshot)

Days 1-30:
- Complete behavior audit and feature/variant inventory
- Build representative corpus and golden baselines
- Draft compatibility matrices (file/embed/browser API)

Days 31-60:
- Freeze contracts
- Deliver state-separation prototype
- Validate deterministic legacy mapping path

Days 61-90:
- Deliver first engine milestone for in-scope features
- Stand up WASM bridge prototype and contract tests

Days 91-120:
- Deliver first end-to-end browser workflow (open/edit/save)
- Run parity gates on initial corpus
- Decide go/no-go for broader variant rollout

---

## 16. Risks and Mitigation

Risk: underestimating compatibility surface (embed params/API)
- Mitigation: freeze compatibility tables early; block deprecations without telemetry evidence.

Risk: state-separation complexity in legacy model
- Mitigation: make Phase 2 mandatory and gate all downstream work on deterministic mapping.

Risk: CAS timeline explosion
- Mitigation: staged CAS delivery; explicit fallback policy; keep CAS off critical path where needed.

Risk: desktop native dependencies increase coupling risk
- Mitigation: defer desktop parity until web runtime and contracts are stable.

Risk: migration fatigue and unclear ownership
- Mitigation: strict checkpoint governance, explicit owner per contract and parity gate.

---

## 17. Practical Definition of Done

Migration is complete only when all are true:
- In-scope documents produce equivalent mathematical behavior.
- Supported variants pass acceptance suites.
- Engine is validated independently from UI runtimes.
- Compatibility commitments are met and documented.
- Platform adapters remain thin and replaceable.
- Legacy stack is removed only after final parity sign-off.

---

## 18. Execution Order (Required)

Execute in this order:
1. Behavior freeze and corpus capture
2. Contract freeze
3. State separation and schema bridge
4. Deterministic engine foundation
5. Browser shell rollout
6. Renderer/editor/keyboard completion
7. Variant expansion
8. Desktop/mobile adapters
9. Legacy removal after hard parity gates

This order turns a high-risk rewrite into measurable, reversible milestones.
