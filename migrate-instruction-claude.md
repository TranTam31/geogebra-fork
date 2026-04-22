# GeoGebra Modern Migration Plan (Claude Specification)

**Status**: Executable Implementation Roadmap  
**Created**: April 2026  
**Primary Focus**: Pragmatic rewrite with risk mitigation and concrete deliverables

---

## Executive Summary

This document refines the original `migrate-instruction.md` by adding:
- Concrete API specifications and schema definitions
- Realistic timeline estimates (18-24 months for Phase 0-3, 3+ years for full migration)
- Team structure requirements
- Risk mitigation strategies
- Detailed sub-phases that fit real project cycles
- Success metrics and go/no-go decision points

**Key Principle**: Build incrementally. Each phase must be shippable and testable independently.

---

## 1. Current State Audit (Phase 0) — 4-6 Weeks

### 1.1 Objectives
- Capture mathematical behavior from legacy system
- Create golden test fixtures
- Build compatibility baseline
- Identify feature-parity gaps

### 1.2 Deliverables

#### 1.2.1 Feature Inventory
Create a comprehensive spreadsheet:

| App Variant | Supported Objects | Supported Commands | Toolbar Count | Unique Features |
|-------------|-------------------|-------------------|---------------|-----------------| 
| Classic | (list) | (list) | N | 3D support |
| Graphing | (list) | (list) | N | Function plotting |
| CAS | (list) | (list) | N | Symbolic algebra |
| Scientific | (list) | (list) | N | Statistical functions |
| ... | ... | ... | ... | ... |

#### 1.2.2 Golden Test Corpus

**Create three test suites**:

a) **Algebra Golden Tests** (stored in `tests/fixtures/algebra/`)
```
- 100 expressions covering: polynomials, trig, calculus, rational functions
- For each: input string → expected simplified form → expected numeric evaluation
- Include edge cases: division by zero, undefined functions, complex numbers
- Format: JSON with versioned schema
```

b) **Geometry Golden Tests** (stored in `tests/fixtures/geometry/`)
```
- 50 construction sequences with expected outputs
- Format: GeoGebra XML (.ggb files) with expected serialized state after load
- Include: point dependencies, dynamic updates, constraints
- Test constructions from 2D, 3D, and geometry-specific app variants
```

c) **CAS Golden Tests** (stored in `tests/fixtures/cas/`)
```
- 75 symbolic operations with expected results
- Include: solve(), factor(), expand(), simplify() and variant-specific functions
- Document Giac version and output format used
```

d) **Document Format Golden Tests**
```
- Real .ggb files from users
- Saved state of each file in the current system (XML dump)
- Document metadata, view settings, object list
```

#### 1.2.3 UI Screenshots and Interactions
- Screenshot of each app variant at startup (10+ variants)
- Screenshots of key workflows (draw object → modify → animate)
- Touch input behavior captures (for mobile planning)
- Keyboard layout captures for scientific/calculator modes

#### 1.2.4 Performance Baseline
Measure and document:
- **Graph update latency**: 100, 500, 1000, 5000 objects → redraw time
- **File load time**: Small (.5MB), Medium (2MB), Large (10MB) .ggb files
- **Memory usage**: Object count vs heap size
- **Expression evaluation**: Complex expression parse + evaluation time

#### 1.2.5 Build System Documentation
- Current Gradle build steps and dependencies
- Build times by subproject
- Known build issues and workarounds
- CI/CD pipeline details

### 1.3 Concrete Tasks

```
Week 1-2: Feature inventory and UI audit
  - List all objects, tools, commands per variant
  - Screenshot each app variant and key workflows
  - Document keyboard layouts (classic, scientific, CAS)
  
Week 2-3: Create golden test corpus
  - Extract 100 algebra expressions from current system
  - Extract 50 geometry constructions (export as .ggb + expected state)
  - Export 75 CAS operations and expected symbolic outputs
  - Collect 20 real user documents for compatibility testing
  
Week 3-4: Performance and build analysis
  - Benchmark graph update latency for 100, 500, 1000, 5000 objects
  - Document file load times for small/medium/large files
  - Analyze current Gradle build, identify bottlenecks
```

### 1.4 Exit Criteria
- ✅ Feature inventory spreadsheet complete and reviewed
- ✅ Golden test corpus stored in version control (algebra, geometry, CAS)
- ✅ Performance baseline documented (latency, memory, build time)
- ✅ At least 20 real .ggb files captured with expected output
- ✅ Team has agreed on feature scope for Phase 2-3

### 1.5 Team: 2-3 engineers (1 testing specialist, 1-2 developers)

---

## 2. API and Schema Design (Pre-Phase 1) — 2-3 Weeks

### 2.1 Critical Specifications Before Any Code

Before writing Rust engine code, document these formally:

#### 2.1.1 Engine API Specification

**Transport**: WASM FFI + JSON over message channel

**Core Operations** (example contract):

```rust
// Pseudo-interface
trait EngineAPI {
    // Document lifecycle
    fn new_document(app_variant: String) -> DocumentHandle;
    fn open_document(bytes: Vec<u8>) -> Result<DocumentHandle, Error>;
    fn save_document(handle: DocumentHandle) -> Vec<u8>;
    
    // Object creation
    fn create_point(doc: DocumentHandle, x: f64, y: f64, name: Option<String>) 
        -> Result<ObjectId, Error>;
    fn create_circle(doc: DocumentHandle, center: ObjectId, radius_or_point: ObjectId) 
        -> Result<ObjectId, Error>;
    fn create_expression(doc: DocumentHandle, text: String) 
        -> Result<ObjectId, Error>;
    
    // Object manipulation
    fn set_property(doc: DocumentHandle, obj: ObjectId, property: String, value: Value) 
        -> Result<(), Error>;
    fn get_property(doc: DocumentHandle, obj: ObjectId, property: String) 
        -> Result<Value, Error>;
    fn delete_object(doc: DocumentHandle, obj: ObjectId) -> Result<(), Error>;
    
    // Dependency tracking and evaluation
    fn evaluate_expression(doc: DocumentHandle, expr: ObjectId) 
        -> Result<Value, Error>;  // Triggers dependent re-evaluation
    fn get_all_objects(doc: DocumentHandle) -> Vec<ObjectMetadata>;
    
    // Undo/redo
    fn undo(doc: DocumentHandle) -> Result<(), Error>;
    fn redo(doc: DocumentHandle) -> Result<(), Error>;
    
    // Selection and views
    fn set_selection(doc: DocumentHandle, objects: Vec<ObjectId>) -> Result<(), Error>;
    fn get_selection(doc: DocumentHandle) -> Vec<ObjectId>;
    
    // Export and serialization
    fn export_to_latex(doc: DocumentHandle, obj: ObjectId) -> String;
    fn export_to_svg(doc: DocumentHandle) -> String;
    fn get_render_commands(doc: DocumentHandle) -> Vec<RenderCommand>;
}

enum Value {
    Number(f64),
    Complex{ real: f64, imag: f64 },
    Vector(Vec<f64>),
    Matrix(Vec<Vec<f64>>),
    List(Vec<Value>),
    String(String),
    Boolean(bool),
    Undefined,
    Error(String),
}

enum RenderCommand {
    DrawPoint { x: f64, y: f64, size: f64, color: String, label: String },
    DrawLine { p1: (f64, f64), p2: (f64, f64), style: LineStyle },
    DrawCurve { points: Vec<(f64, f64)>, color: String },
    DrawText { x: f64, y: f64, text: String, font_size: f64 },
}
```

**Versioning**: 
- Engine API version is semantic (Major.Minor.Patch)
- Breaking changes only on Major version bump
- Document changelog per version

#### 2.1.2 Document Schema Specification

**Format**: JSON (with optional binary compression)

**Document v2.0 Schema**:

```json
{
  "version": "2.0",
  "app_variant": "graphing",  // classic, graphing, 3d, cas, scientific, geometry, etc.
  "metadata": {
    "title": "My Construction",
    "created": "2026-04-22T10:30:00Z",
    "modified": "2026-04-22T15:45:00Z",
    "author": "User Name",
    "description": ""
  },
  "settings": {
    "axes_visible": true,
    "grid_visible": true,
    "view_scale": 1.0,
    "view_offset_x": 0,
    "view_offset_y": 0,
    "angle_unit": "degrees",  // degrees, radians
    "decimal_places": 2
  },
  "objects": [
    {
      "id": "obj_001",
      "type": "point",  // point, line, circle, function, expression, etc.
      "name": "A",
      "definition": {
        "x": 1.5,
        "y": 2.3
      },
      "properties": {
        "visible": true,
        "color": "#FF0000",
        "size": 5,
        "label_visible": true
      },
      "dependent_on": []
    },
    {
      "id": "obj_002",
      "type": "point",
      "name": "B",
      "definition": {
        "x": 4.0,
        "y": 1.0
      },
      "properties": {
        "visible": true,
        "color": "#0000FF",
        "size": 5,
        "label_visible": true
      },
      "dependent_on": []
    },
    {
      "id": "obj_003",
      "type": "line",
      "name": "line1",
      "definition": {
        "through_points": ["obj_001", "obj_002"]
      },
      "properties": {
        "visible": true,
        "color": "#000000",
        "thickness": 1
      },
      "dependent_on": ["obj_001", "obj_002"]
    },
    {
      "id": "obj_004",
      "type": "expression",
      "name": "f(x)",
      "definition": {
        "text": "x^2 + 2*x + 1"
      },
      "properties": {
        "visible": true
      },
      "dependent_on": []
    }
  ],
  "views": {
    "algebra": {
      "visible": true,
      "width_percent": 30
    },
    "graphics": {
      "visible": true,
      "width_percent": 70
    },
    "table": {
      "visible": false
    }
  },
  "selections": {
    "current": []
  },
  "animation": {
    "speed": 1.0,
    "is_playing": false
  }
}
```

**Compatibility Strategy**:
- Old .ggb files (v1.0) converted to v2.0 on import (with translation layer)
- Unsupported v1.0 features flagged at import time
- Document can be marked "compatibility mode" if legacy features present

#### 2.1.3 Expression Parser Grammar

**Goal**: Define the subset of GeoGebra syntax that v1.0 must support

**BNF Grammar Sketch**:

```
expression ::= logical_or
logical_or ::= logical_and ( "or" logical_and )*
logical_and ::= comparison ( "and" comparison )*
comparison ::= additive ( ( "==" | "!=" | "<" | ">" | "<=" | ">=" ) additive )*
additive ::= multiplicative ( ( "+" | "-" ) multiplicative )*
multiplicative ::= power ( ( "*" | "/" | "^" ) power )*
power ::= primary ( "^" primary )?
primary ::= 
    | NUMBER
    | VARIABLE
    | FUNCTION "(" arguments ")"
    | "(" expression ")"
    | matrix_literal

arguments ::= expression ( "," expression )*

matrix_literal ::= "{" row ( "," row )* "}"
row ::= "{" expression ( "," expression )* "}"
```

**Supported Functions** (Phase 2a):
- Math: sin, cos, tan, sqrt, abs, log, ln, exp, min, max, round, floor, ceiling
- Vector: norm, dot, cross
- Matrix: determinant, transpose, inverse
- Algebra: factor, expand, simplify, solve

**Execution Model**:
- Expressions are lazy-evaluated
- Dependencies trigger automatic re-evaluation
- Circular dependencies are detected and reported as errors

#### 2.1.4 Platform Adapter Contract

**Interface Each Platform Must Implement**:

```typescript
// Pseudo-TypeScript interface for web adaptation
interface PlatformAdapter {
  // File I/O
  readFile(path: string): Promise<ArrayBuffer>;
  writeFile(path: string, data: ArrayBuffer): Promise<void>;
  
  // Clipboard
  copyToClipboard(text: string): Promise<void>;
  readFromClipboard(): Promise<string>;
  
  // Rendering
  getCanvasContext(): CanvasRenderingContext2D | WebGLContext;
  requestAnimationFrame(callback: () => void): void;
  
  // Input
  onKeyDown(handler: (key: string, modifiers: KeyModifiers) => void): void;
  onMouseMove(handler: (x: number, y: number) => void): void;
  onTouchEvent(handler: (touches: Touch[]) => void): void;
  
  // Localization
  getString(key: string, lang: string): string;
  
  // System info
  getDevicePixelRatio(): number;
  getScreenSize(): { width: number, height: number };
}
```

### 2.2 Exit Criteria
- ✅ Engine API specification reviewed and approved
- ✅ Document schema (v2.0) finalized with examples
- ✅ Expression grammar defined and tested against golden corpus
- ✅ Platform adapter interface specified
- ✅ WASM FFI boundary documented (message format, error handling)
- ✅ Versioning policy established (backward compatibility rules)

### 2.3 Team: 1-2 architects

---

## 3. Rust Engine Core (Phase 2) — 6-12 Months

### 3.1 Sub-phases

#### Phase 2a: Core Object Model (6-8 Weeks)
**Deliverable**: Can create and serialize basic objects

**Tasks**:
- Set up Rust workspace, FFI bindings, wasm compilation pipeline
- Implement object representation (Point, Line, Circle, Polygon, Expression)
- Implement property storage and retrieval
- Implement serialization to JSON
- Write unit tests

**Exit criteria**:
- Can create Point, Line, Circle; serialize to JSON schema
- Unit tests pass; golden tests for basic objects pass

#### Phase 2b: Expression Evaluation (8-10 Weeks)
**Deliverable**: Parse and evaluate mathematical expressions

**Tasks**:
- Implement expression parser from grammar spec
- Implement numeric evaluator (arithmetic, functions, variables)
- Implement symbolic algebra hooks (placeholder for CAS integration)
- Implement automatic re-evaluation on dependency change
- Handle errors gracefully

**Exit criteria**:
- All 100 golden algebra tests pass
- 50/75 CAS tests pass (basic operations without symbolic algebra)

#### Phase 2c: Dependency Tracking and Undo/Redo (6-8 Weeks)
**Deliverable**: Objects track dependencies; undo/redo works

**Tasks**:
- Implement dependency graph
- Implement change notifications
- Implement undo/redo command stack
- Test circular dependency detection
- Performance test with 1000+ objects

**Exit criteria**:
- Undo/redo works through 50+ operations
- Dependency graph correctly propagates changes
- No memory leaks with 5000 objects

#### Phase 2d: Document Serialization (4-6 Weeks)
**Deliverable**: Load and save complete documents

**Tasks**:
- Implement v2.0 document schema serialization
- Implement v1.0 (legacy .ggb) format reader
- Build compatibility layer for old object types
- Test round-tripping: load → modify → save → load

**Exit criteria**:
- All 20 golden .ggb test files load correctly
- Modified documents save and re-load with identical state
- Unsupported v1.0 features are logged and optional

#### Phase 2e: CAS Integration (4-6 Weeks)
**Deliverable**: Symbolic algebra operations work

**Tasks**:
- Define CAS service interface
- Integrate with existing Giac JNI or port key functions to Rust
- Implement symbolic evaluation in expression evaluator
- Handle CAS errors gracefully

**Exit criteria**:
- 70+ golden CAS tests pass
- solve(), factor(), expand(), simplify() work
- Fallback behavior when CAS unavailable

#### Phase 2f: Render Commands (4-6 Weeks)
**Deliverable**: Engine exports render commands, not rendered pixels

**Tasks**:
- Define RenderCommand enum
- Implement scene description from object state
- Export 2D geometry drawing commands
- Include text, labels, annotations

**Exit criteria**:
- RenderCommand output validated against golden screenshots
- Coordinate system transforms work correctly

### 3.2 Team: 4-6 Rust engineers

### 3.3 Success Metrics
- Test coverage: > 85% for engine code
- All golden tests pass (algebra, geometry, document round-trip)
- Performance: Expression evaluation < 10ms, 1000-object graph update < 50ms
- Zero memory leaks or dangling pointers
- WASM binary size < 10MB

---

## 4. Web Browser Shell (Phase 3) — 4-6 Months

### 4.1 Sub-phases

#### Phase 3a: Web Build System and WASM Integration (3 weeks)
**Deliverable**: TypeScript + React project that loads engine via WASM

**Tasks**:
- Set up TypeScript + Vite + React project
- Configure WASM compilation and bundling
- Implement FFI bridge layer (Rust ↔ TypeScript)
- Set up CI/CD pipeline for web builds
- Test WASM binary loads and initializes

**Exit criteria**:
- Web dev server runs; WASM engine loads in < 2 seconds
- FFI bridge tested with round-trip message passing

#### Phase 3b: Core UI Shell (3-4 weeks)
**Deliverable**: App variants can launch; basic layout works

**Tasks**:
- Build React component hierarchy (AppShell, Toolbar, Canvas, AlgebraView)
- Implement variant routing (classic, graphing, 3d, etc.)
- Implement document open/new/save workflows
- Set up state management (Redux or Zustand)
- Add basic settings and preferences UI

**Exit criteria**:
- Each app variant launches without errors
- Canvas and algebra view render placeholders
- Document save/load workflow functional

#### Phase 3c: Canvas Rendering Backend (4-5 weeks)
**Deliverable**: Objects render correctly in canvas

**Tasks**:
- Consume RenderCommand from engine
- Implement Canvas 2D rendering for: points, lines, circles, curves
- Implement text/label rendering
- Add dynamic coordinate transformations (zoom, pan)
- Implement hit testing for mouse interaction

**Exit criteria**:
- Golden construction screenshots match reference (< 5% visual diff)
- Zoom/pan/drag interactions work smoothly
- Performance: 60+ FPS for 100-object graph

#### Phase 3d: Math Input and Keyboard (4 weeks)
**Deliverable**: Users can enter expressions and commands

**Tasks**:
- Implement formula input field with syntax highlighting
- Build virtual keyboard component (reuse from shared schema)
- Hook keyboard actions to engine object creation
- Implement autocomplete and symbol insertion
- Add accessibility support (ARIA labels)

**Exit criteria**:
- Can type expression, press Enter, object appears on canvas
- Scientific and calculator keyboard variants work
- Keyboard is touch-friendly (40x40px minimum tap target)

#### Phase 3e: Animation and Interaction (3-4 weeks)
**Deliverable**: Can drag points, animate constructions, interact smoothly

**Tasks**:
- Implement point dragging and real-time updates
- Implement slider animation controls
- Connect tool system (draw circle, draw line tools)
- Implement property panels for object editing
- Test interaction responsiveness

**Exit criteria**:
- Drag a point → dependent objects update smoothly
- Animation controls work; animation at 30+ FPS with 500 objects
- Key workflows executable: draw → constrain → animate

### 4.2 Team: 3-4 React engineers + 1 DevOps

### 4.3 Success Metrics
- Web app loads in < 3 seconds on 4G connection
- Canvas rendering: 60+ FPS for 100-object constructions
- All golden workflows executable in browser
- Cross-browser tested (Chrome, Firefox, Safari)
- Mobile browser support (responsive design)
- Accessibility: WCAG 2.1 Level AA


### 4.4 Deliverable: Shippable Beta
- Production build deployed to beta.geogebra.org
- Feature parity with legacy for: Graphing, Scientific, Classic (basic)
- User feedback collected; known limitations documented
- Telemetry and error logging in place

---

## 5. Renderer and View Layers (Phase 4) — 8-10 Weeks

### 5.1 Objectives
- Add algebra view, table view
- Support 3D rendering (via Three.js or Babylon.js, not JOGL)
- Achieve visual parity with legacy for key constructions
- Optimize performance

### 5.2 Sub-phases

#### Phase 4a: Lightweight Renderer Abstraction (3 weeks)
**Deliverable**: Switch between Canvas, WebGL, SVG backends without changing engine

**Task**:
- Define renderer interface in engine
- Implement Canvas 2D backend (primary for web)
- Implement SVG backend (for export)
- Add feature detection (WebGL availability)

#### Phase 4b: Algebra View (2-3 weeks)
**Deliverable**: Display object definitions and values alongside graphics

**Tasks**:
- Consume object metadata from engine
- Render in tree or list format
- Support editing properties (color, visibility, etc.)
- Refresh on engine state change

#### Phase 4c: 3D View (4-5 weeks)
**Deliverable**: Render 3D objects (if product includes 3D variant, else skip for v1)

**Decision Point**: Is 3D a must-have for v1.0?
- If **yes**: Implement Three.js backend for 3D surface, point, line visualization
- If **no**: Scope for Phase 5 (post launch)

#### Phase 4d: Table and Statistics Views (2-3 weeks)
**Deliverable**: Display list data and computed statistics

#### Phase 4e: Export to Image/PDF (2 weeks)
**Deliverable**: Save construction as SVG, PNG, PDF

### 5.3 Team: 2-3 engineers

### 5.4 Exit Criteria
- Golden screenshot tests pass (algebra, graphics, 3D if included)
- Performance maintained (60 FPS for complex constructions)
- Export formats validated

---

## 6. Editor and Keyboard Refinement (Phase 5) — 6-8 Weeks

### 6.1 Objectives
- Enhance formula editor with autocomplete, syntax highlighting, error recovery
- Optimize keyboard interaction for all input methods
- Accessibility compliance

### 6.2 Deliverables
- Rich text editor with live preview
- Autocomplete with function signatures
- Symbol insertion and LaTeX preview
- Touch keyboard optimized for mobile
- Screen reader support

### 6.3 Team: 2 engineers

---

## 7. App Variants (Phase 6) — 6-8 Weeks

### 7.1 Variants and Feature Flags

Each variant is configured via startup flags:

```json
{
  "graphing": {
    "tools": ["move", "point", "line", "circle", "polygon", "function"],
    "views": ["algebra", "graphics"],
    "functions": ["sin", "cos", "exp", "log"],
    "allow_3d": false
  },
  "scientific": {
    "tools": ["calculator"],
    "views": ["algebra"],
    "functions": ["all"],
    "allow_complex": true,
    "keyboard": "scientific"
  },
  "cas": {
    "tools": ["calculator"],
    "views": ["algebra"],
    "allow_symbolic": true,
    "keyboard": "cas"
  },
  "3d": {
    "tools": ["move", "point", "line", "sphere", "surface"],
    "views": ["algebra", "graphics_3d"]
  }
}
```

### 7.2 Deliverables
- Feature flag system
- Variant-specific startup configurations
- Variant-specific toolbar and keyboard layouts
- Variant-specific documentation

### 7.3 Team: 1-2 engineers + QA

---

## 8. Desktop Shell and Mobile Adapters (Phase 7) — 12+ Weeks

### 8.1 Decision Point: Desktop Framework

Choose **ONE**:
- **Option A**: Tauri (lightweight, Rust-friendly, cross-platform)
- **Option B**: Electron (heavier, but more mature)
- **Option C**: Native desktop shell per platform (SwiftUI for macOS, WinUI for Windows)

**Recommendation**: Start with **Option A (Tauri)** to minimize initial complexity

### 8.2 Mobile Apps

**iOS**: SwiftUI + shared Rust engine via FFI  
**Android**: Jetpack Compose + shared Rust engine via FFI

**Key Challenge**: Mobile UX is fundamentally different
- Touch gestures instead of mouse
- Smaller viewport
- Mobile-optimized keyboard
- Platform conventions (iOS vs Android)

**Recommendation**: Treat mobile as a Design Phase 7a before development

### 8.3 Timeline
- Desktop (Tauri): 6-8 weeks
- iOS: 8-10 weeks
- Android: 8-10 weeks

### 8.4 Team: 2-3 desktop engineers + 1-2 iOS engineers + 1-2 Android engineers

---

## 9. Legacy File Compatibility (Phase 8) — 4-6 Weeks

### 9.1 Objectives
- Support importing old .ggb files
- Map v1.0 object types to v2.0
- Flag unsupported features

### 9.2 Compatibility Rules

```
v1.0 Object Type  →  v2.0 Equivalent  →  Supported?
Point             →  Point            →  Yes
Line              →  Line             →  Yes
Circle            →  Circle           →  Yes
Polygon           →  Polygon          →  Yes
Function          →  Expression       →  Yes
Slider            →  Slider           →  Phase 9
Locus              →  Path             →  Phase 9
Constraint        →  Constraint       →  Partial
Image             →  Image            →  No (Phase 10)
Text              →  Text             →  Yes
Button            →  Button           →  No (Phase 10)
```

### 9.3 Deliverables
- XML parser for legacy .ggb format
- Object type translator
- Compatibility logger (what was/wasn't converted)
- Test suite with 20+ real legacy documents

### 9.4 Team: 1-2 engineers

---

## 10. Testing and Quality Assurance — Every Phase

### 10.1 Golden Test Suite

**Maintained throughout all phases**:

```
tests/
  fixtures/
    algebra/
      expressions.json (100 test cases)
      expected_outputs.json
    geometry/
      constructions/ (50 .ggb files)
      expected_states.json
    cas/
      symbolic.json (75 test cases)
    legacy/
      real_documents/ (20 .ggb files)
      expected_outputs.json
  engine/
    unit_tests/ (Rust)
  web/
    integration_tests/ (Cypress)
    performance_tests/
    accessibility_tests/
  golden_screenshots/
    graphing/
      construction_1.png
      construction_1_ref.png (reference)
      diff_report.html
```

### 10.2 CI/CD Pipeline

```yaml
# Every commit
- Lint (Rust: clippy, TypeScript: eslint)
- Build (Rust WASM, TypeScript bundle)
- Unit tests (Rust engine tests)
- Golden tests (algebra, geometry, CAS)

# Daily
- Integration tests (web app end-to-end)
- Performance tests (render latency, startup time)
- Screenshot diff tests

# Weekly
- Accessibility audit
- Compatibility test (legacy documents)
- Cross-browser testing

# Monthly
- Security scanning
- Dependency updates
- Load testing (1000+ concurrent users)
```

### 10.3 Performance Benchmarks

| Metric | Requirement | Failure Threshold |
|--------|-------------|-------------------|
| **Engine startup** | < 100ms | > 200ms |
| **Point creation** | < 5ms | > 15ms |
| **1000-object update** | < 50ms | > 100ms |
| **Canvas render** | 60 FPS | < 30 FPS |
| **WASM load** | < 2s on 4G | > 5s |
| **Memory per 1000 objects** | < 50MB | > 100MB |

---

## 11. Risk Mitigation

### 11.1 High-Risk Areas

| Risk | Mitigation |
|------|-----------|
| **Rust learning curve** | Pair with experienced Rust dev; code reviews for first 3 months |
| **Parser semantics differ from legacy** | Phase 0 golden tests catch this early; slow down Phase 2b if needed |
| **WASM FFI performance issues** | Prototype FFI in Phase 3a; profile early; batch API calls |
| **3D rendering performance** | Defer to Phase 4c; assess Three.js cost in spike study |
| **CAS integration complexity** | Investigate Giac FFI in spike; consider pure Rust fallback |
| **Mobile UX mismatch** | Conduct usability study in Phase 7a; prototype touch gestures |
| **Undo/redo correctness** | Extensive property-based testing; fuzzing |
| **Desktop file dialogs** | Use standard platform APIs; test early |

### 11.2 Go/No-Go Decision Points

**After Phase 2 (8-12 weeks)**:
- ✅ Can engine reproduce 85%+ of golden tests?
- ✅ Are undo/redo and dependencies correct?
- ✅ Performance acceptable (< 10ms expression eval)?
- **Decision**: Proceed to Phase 3 or pivot to TypeScript engine?

**After Phase 3 (4-6 months)**:
- ✅ Is web app launching and responsive?
- ✅ Can users create and save documents?
- ✅ Golden workflows mostly functional?
- **Decision**: Beta launch for user testing or extend Phase 3?

---

## 12. Team and Timeline Summary

### 12.1 Staffing Model

| Phase | Duration | Core Team | Specialists | Total |
|-------|----------|-----------|-------------|-------|
| Phase 0 | 4-6 weeks | 2-3 | 1 QA | 3 FTE |
| Phase 1 | 2-3 weeks | 1-2 | - | 2 FTE |
| Phase 2 | 6-12 months | 4-6 Rust | 1 DevOps | 7 FTE |
| Phase 3 | 4-6 months | 3-4 React | 1 DevOps | 5 FTE |
| Phase 4 | 8-10 weeks | 2-3 | - | 3 FTE |
| Phase 5 | 6-8 weeks | 2 | - | 2 FTE |
| Phase 6 | 6-8 weeks | 1-2 | 2 QA | 4 FTE |
| Phase 7 | 12+ weeks | 2-3 desktop, 1-2 mobile per platform | 1 DevOps | 8 FTE |
| Phase 8 | 4-6 weeks | 1-2 | - | 2 FTE |

**Overlapping Staffing**: Many phases run in parallel (Phase 2 + Phase 3 overlap, etc.)

**Critical Path**: Phase 0 → Phase 1 → Phase 2 (sequential)  
**Parallelizable**: Phase 3, 4, 5 can happen in parallel once Phase 2 reaches v1.0

### 12.2 Total Timeline (Best Case, Parallel Work)

- **Months 1-2**: Phase 0-1 (Audit + API Design)
- **Months 2-8**: Phase 2 (Rust Engine) + Phase 3 (Web) overlap
- **Months 8-12**: Phase 4-5 (Renderers + Editor) + Phase 6 (Variants) parallel
- **Months 12-18**: Phase 7 (Desktop/Mobile)
- **Months 18-20**: Phase 8 (Legacy Compatibility)
- **Month 20**: Phase 9 (Cleanup and full launch)

**Total**: **18-24 months for core rewrite** + **3-6 months for stabilization/launch**

---

## 13. Minimal Viable Product (MVP) Definition

### What v1.0 Must Include
- ✅ Rust engine with expression parsing and evaluation
- ✅ Web browser shell (TypeScript + React)
- ✅ Graphing app variant (primary focus)
- ✅ Canvas rendering (2D)
- ✅ Formula input and keyboard
- ✅ Document save/load (new .ggb-v2.0 format)
- ✅ Basic undo/redo
- ✅ Zoom, pan, drag interaction

### What v1.0 Can Defer (v1.1, v2.0)
- ❌ Scientific calculator variant (Phase 6 extension)
- ❌ CAS variant (Phase 6 extension)
- ❌ 3D rendering (Phase 4 decision)
- ❌ Mobile apps (Phase 7)
- ❌ Desktop app (Phase 7; web-only for v1.0)
- ❌ Legacy .ggb import (Phase 8)
- ❌ Real-time collaboration
- ❌ Custom scripting API

### MVP Success Criteria
1. **Parity Testing**: Reference set of 10 complex constructions work identically in old and new
2. **Performance**: Startup < 2s, graph update < 100ms for 500 objects
3. **User Testing**: 10+ internal users can build constructions without documentation
4. **Stability**: 99.5% uptime; < 1 crash per 100 sessions
5. **Browser Support**: Chrome, Firefox, Safari on desktop; Chrome on Android

---

## 14. What To Keep vs. Replace

### Keep
- ✅ Mathematical engine logic (translate to Rust)
- ✅ Document structure concept (port to JSON)
- ✅ App variant framework
- ✅ Keyboard separation model
- ✅ Tool and object taxonomy
- ✅ Golden test principle
- ✅ Platform adapter pattern

### Replace
- ❌ GWT compilation and bootstrap
- ❌ Gradle monorepo (switch to Cargo + Yarn)
- ❌ Java-based UI layers (build React)
- ❌ Browser-specific GWT code generation
- ❌ Old .ggb XML format (switch to JSON-v2.0)
- ❌ Hard coupling between engine and UI
- ❌ JVM-only build assumptions

---

## 15. Success Metrics and Retrospective

### Phase Completion Checklist

**Phase 0 (Audit)**:
- [ ] Feature inventory complete
- [ ] 100 golden algebra tests captured
- [ ] 50 golden geometry constructions captured
- [ ] 75 golden CAS tests captured
- [ ] 20 real legacy documents archived
- [ ] Performance baseline measured

**Phase 2 (Engine)**:
- [ ] 85%+ unit test coverage
- [ ] All golden tests pass
- [ ] WASM binary < 10MB
- [ ] Expression eval < 10ms
- [ ] 1000-object update < 50ms
- [ ] Zero memory leaks (Valgrind/MIRI)

**Phase 3 (Web Shell)**:
- [ ] Production build < 5MB gzipped
- [ ] Startup < 3s on 4G
- [ ] Canvas render 60+ FPS for 100 objects
- [ ] All major workflows functional
- [ ] Cross-browser tested
- [ ] 95+ Lighthouse score

**Post-Launch (Ongoing)**:
- [ ] User feedback incorporated
- [ ] Performance improvements
- [ ] Bug fixes prioritized
- [ ] Scaling to mobile/desktop

---

## 16. Escalation and Decision Framework

### Escalation Triggers

| Situation | Action |
|-----------|--------|
| **Golden test pass rate < 80%** | Halt progress; deep dive on failing tests |
| **WASM binary > 15MB** | Review code size; consider modularization |
| **Expression eval > 20ms** | Profile; optimize hot paths or defer features |
| **Canvas FPS < 30 for 100 objects** | Review rendering; consider WebGL optimization |
| **CAS integration blocka** | Spike on Giac FFI; prepare Rust fallback |
| **Team onboarding > 6 weeks** | Reduce scope or extend timeline |

### Decision Gates

| Gate | Criteria |
|------|----------|
| **End of Phase 0**: Proceed? | Feature inventory complete; team aligned |
| **Mid Phase 2**: Proceed? | Object model working; tests passing |
| **End of Phase 2**: Proceed to Phase 3? | Engine  85%+ correct; performance OK |
| **Mid Phase 3**: Proceed? | Web shell renders UI; WASM FFI working |
| **End of Phase 3**: Beta launch? | MVP workflows functional; < 100 known issues |

---

## 17. Dependencies and Build System

### New Repo Structure

```
geogebra-next/
├── crates/
│   ├── engine/              # Core math engine
│   │   ├── src/
│   │   ├── tests/
│   │   └── Cargo.toml
│   ├── parser/              # Expression parser
│   ├── cas-bridge/          # CAS integration
│   ├── wasm-ffi/            # WASM interface
│   └── document/            # Document model
├── apps/
│   ├── web/                 # React TypeScript web app
│   │   ├── src/
│   │   ├── vite.config.ts
│   │   └── package.json
│   ├── desktop/             # Tauri desktop shell (Phase 7)
│   ├── ios/                 # SwiftUI app (Phase 7)
│   └── android/             # Compose app (Phase 7)
├── tests/
│   ├── fixtures/            # Golden test data
│   ├── e2e/                 # Cypress tests
│   └── performance/         # Benchmark suite
├── docs/
│   ├── api-spec.md
│   ├── schema-v2.0.md
│   └── architecture.md
├── tooling/
│   ├── codegen/
│   └── migrations/
├── Cargo.toml              # Root workspace
└── package.json            # Monorepo root (Yarn workspaces)
```

### Build Tool Recommendations

- **Rust**: Cargo (standard)
- **TypeScript/Web**: Vite + pnpm
- **CI/CD**: GitHub Actions or GitLab CI
- **Docker**: For consistent build environments
- **Version Control**: Git; monorepo strategy (Cargo workspaces + pnpm)

---

## 18. Final Recommendations

### Start Immediately (Phase 0)
1. Execute feature audit and create golden test corpus NOW
2. This de-risks the entire rewrite; without it, you're guessing
3. Allocate 4-6 weeks; hire or redirect 2-3 engineers

### Before Writing Engine Code (Phase 1)
1. Finalize API and schema specifications
2. Prototype WASM FFI bridge (small spike, 1 week)
3. Get team trained on Rust (async, ownership, borrowing)
4. Set up CI/CD infrastructure

### Engine Development (Phase 2)
1. Use test-driven development; golden tests are your spec
2. Profile early and often; performance regression is expensive to fix later
3. Avoid feature creep; cut anything not in v1.0 scope
4. Consider pair programming for complex algorithms

### Launch Strategy
1. Web (Phases 2-3) → Graphing app variant → Beta launch
2. Desktop/Mobile (Phase 7) → after web is stable
3. Variants (Phase 6) → add CAS, Scientific after Graphing ships

### Expect Challenges
- Parser correctness issues (6-8 weeks debugging)
- Undo/redo complexity (subtle bugs, easy to miss)
- CAS integration pain (library incompatibility, version hell)
- Mobile UX redesign (bigger than expected)
- Performance surprises (WASM startup, memory overhead)

---

## 19. Conclusion

This migration is **feasible but non-trivial**. Success requires:

1. **Discipline**: DO NOT skip Phase 0 or Phase 1; golden tests and API design save months
2. **Realistic timeline**: 18-24 months minimum for core rewrite
3. **Staged rollout**: Ship web first, mobile/desktop later
4. **Risk management**: Go/no-go decision points after each major phase
5. **Team investment**: Rust expertise is non-negotiable for Phase 2

**Recommendation**: Begin with Phase 0 (audit and freeze behavior) within 2 weeks. This decision point determines if the rewrite is actually viable or if modifications to the current stack are more pragmatic.

---

## Appendix: Example Milestones for Executive Reporting

| Milestone | Target Date | Key Deliverable | Success Criteria |
|-----------|-------------|-----------------|------------------|
| **Phase 0 Complete** | Week 6 | Golden test corpus, feature inventory | 100+ algebra tests, 50 geometry constructions |
| **API Spec Approved** | Week 9 | Engine and schema frozen | No changes to API signatures for Phase 2 |
| **Object Model Working** | Week 15 | Basic point, line, circle creation | Can create and serialize 100 objects |
| **Expression Eval Working** | Week 25 | Parsing and numeric evaluation | All golden algebra tests pass |
| **Engine v1.0 Complete** | Month 8 | Ready for WASM binding | 85%+ test pass rate |
| **Web Shell Alpha** | Month 10 | Browser app loads graphing variant | Canvas renders, can create points |
| **Rendering Parity** | Month 12 | Screenshots match golden images | Visual diff < 5% for 10 constructions |
| **Web Shell Beta Launch** | Month 14 | Public beta for graphing | < 100 known issues, 99.5% uptime |
| **Desktop/Mobile Alpha** | Month 16 | Tauri/iOS/Android shells working | Cross-platform basic workflows test |
| **Post-Migration Cleanup** | Month 20 | Old code archived, new stack shipping | Zero dependencies on old GWT/Java UI |

---

**End of Document**  
*Authored: April 2026*  
*Status: Ready for Phase 0 execution*
