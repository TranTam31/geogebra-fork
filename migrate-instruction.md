# GeoGebra Modern Migration Instruction

This document is a practical migration blueprint for turning the current GeoGebra monorepo into a modern product stack while preserving the mathematical behavior, app variants, and cross-platform reach.

The goal is not to rewrite everything at once. The goal is to replace the architecture in layers so that the new codebase can prove correctness module by module.

## 1. What This Repo Is Today

The current repository is a large Gradle composite build with three major platform families:

- Shared core logic in source/shared
- Web application in source/web
- Desktop application in source/desktop

The current web app is not a Node or React frontend. It is a GWT-based Java-to-JavaScript application with HTML templates, generated assets, and a dev server that can run in a browser at port 8888.

The current desktop app is a JVM application with native OpenGL and Giac bindings.

The main architectural fact that drives the migration is this:

- The mathematical engine is already separated from the UI in many places, but the code is still organized around old platform-specific assumptions.
- The browser UI, desktop UI, formula editor, keyboard, renderer, and native CAS all depend on each other through a large shared Java codebase.
- A modern rewrite must preserve the engine contract while replacing the presentation and platform layers.

## 2. Migration Strategy in One Sentence

Build a new repository around a small, strict core engine, then rebuild each user-facing surface as a thin platform adapter around that engine.

The recommended target stack is:

- Core engine and geometry kernel: Rust
- Browser UI: TypeScript + React
- Browser rendering runtime: WebAssembly for engine execution, Canvas/SVG/WebGL for drawing
- Desktop shell: native desktop shell or Tauri/Electron only as a temporary bridge
- Mobile apps: native SwiftUI for iOS and Jetpack Compose for Android, both using the shared Rust engine
- Shared protocol layer: versioned JSON and binary schemas generated from one source of truth
- Build and orchestration: modern package manager and task runner, plus CI pipelines that can test each package independently

If you want to reduce risk even more, the very first new repo can be Rust core + TypeScript web only, while desktop and mobile stay on the old stack until the engine stabilizes.

## 3. Non-Negotiable Migration Rules

These rules should be treated as architectural law during the rewrite.

1. Do not port UI code before engine behavior is covered by tests.
2. Do not copy old architecture into the new repo just because it exists today.
3. Do not allow the new UI to call engine internals directly. Use a stable API boundary.
4. Do not merge all platforms into one shared runtime again.
5. Do not replace math behavior without golden tests and sample files.
6. Do not begin with visuals or theming. Begin with formulas, objects, coordinate transforms, and serialization.
7. Do not assume all current features belong in v1 of the new stack.

## 4. Current Module Inventory and What Each One Must Become

This section maps the existing repository modules to their future role.

### 4.1 Shared Foundation

#### source/shared/common
Current role:
- Core GeoGebra domain logic
- Construction of objects, expressions, algebra, geometry, CAS behavior, and application state
- The most important source of product truth

New role:
- Become the reference behavior for the new Rust engine
- Every important class or subsystem here must be translated into an engine capability, API, or test fixture
- Any UI coupling must be removed or isolated

What the new version needs to do:
- Evaluate expressions
- Represent geometric objects and dependencies
- Maintain undo/redo history
- Serialize and deserialize documents
- Manage selections, views, tools, and modes
- Compute transformations, constraints, and dynamic updates
- Expose stable APIs for all platforms

Migration output:
- Rust crate for kernel and object model
- Shared test corpus of documents, expressions, and expected outputs
- Formal API boundaries for geometry, algebra, statistics, and CAS behavior

#### source/shared/common-jre
Current role:
- JVM-specific support for common logic
- Headless runtime helpers, tests, and scripting integration

New role:
- Become either a thin JVM compatibility bridge or disappear entirely
- Any behavior used only by JVM tests should be replaced by platform-neutral tests or by a dedicated test harness around the new Rust engine

What the new version needs to do:
- Provide JVM test adapters only if they are still needed
- Avoid becoming the place where platform-specific business logic accumulates

Migration output:
- Replace with engine integration tests and a lightweight JVM bridge layer only if necessary

#### source/shared/editor-base
Current role:
- Math editor abstractions and parser/editor support

New role:
- Become the canonical editor grammar, AST, and editing commands shared by all frontends

What the new version needs to do:
- Parse math input into a typed syntax tree
- Support inline editing, cursor movement, autocomplete, and command insertion
- Support LaTeX-like and GeoGebra-specific representations
- Provide error recovery and incremental parsing

Migration output:
- Shared parser crate or package
- Stable syntax tree format
- Editor command protocol

#### source/shared/renderer-base
Current role:
- Shared rendering primitives and math typesetting foundation

New role:
- Become the rendering contract, not the rendering implementation

What the new version needs to do:
- Describe formulas, text layout, and vector drawing output in platform-neutral terms
- Define how a formula or object is measured, laid out, and rendered
- Expose text metrics and glyph resolution interfaces

Migration output:
- A renderer protocol layer
- Reference layout tests for formulas and labels
- Platform renderers for browser and desktop

#### source/shared/canvas-base
Current role:
- Common canvas abstraction

New role:
- Become the low-level drawing contract used by all renderers

What the new version needs to do:
- Represent 2D drawing commands
- Support paths, strokes, fills, text, images, clipping, transforms, and hit testing
- Provide a bridge to Canvas 2D, SVG, Metal, Skia, or another platform renderer

Migration output:
- Small rendering command language
- Cross-platform rendering test harness

#### source/shared/keyboard-base
Current role:
- Keyboard model and layouts

New role:
- Become the shared keyboard schema and interaction model

What the new version needs to do:
- Define layout rows, keys, actions, variants, and language-specific labels
- Support math keys, symbols, navigation, and function insertion
- Separate keyboard data from keyboard rendering

Migration output:
- Shared keyboard schema
- Frontend-agnostic key action protocol

#### source/shared/keyboard-scientific
Current role:
- Scientific calculator keyboard extensions

New role:
- Become a variant package inside the new keyboard system

What the new version needs to do:
- Add scientific-only symbols, function keys, and shortcuts
- Reuse the same keyboard data model as the rest of the product

Migration output:
- Variant keyboard definition file or package

#### source/shared/giac-jni
Current role:
- JNI binding to Giac for CAS behavior

New role:
- Move CAS integration behind a clean engine service boundary

What the new version needs to do:
- Provide symbolic algebra operations through a stable interface
- Hide native library loading and platform differences from the UI
- Allow fallback behavior when CAS is unavailable

Migration output:
- CAS service interface
- Native bridge layer isolated from UI code
- Test cases for CAS commands and expression transformations

#### source/shared/xr-base
Current role:
- Extended reality base layer

New role:
- Either preserve as an experimental platform plugin or leave it out of the first rewrite wave

What the new version needs to do:
- Define optional 3D/XR interaction and rendering hooks only if the product plan includes them

Migration output:
- Keep out of v1 unless there is a real product requirement

#### source/shared/ggbjdk
Current role:
- Shared JDK-related support package

New role:
- Convert into either a compatibility test package or a support library for migration tooling

What the new version needs to do:
- Support tests and utilities that help verify engine compatibility

Migration output:
- Keep only if test utilities remain useful in the new stack

### 4.2 Web Layer

#### source/web/web
Current role:
- The main web orchestrator
- GWT module compilation
- HTML generation
- CSS compilation
- Dev-mode server
- Multi-app browser bootstrapping

New role:
- Become the specification for what the browser product must do, not the implementation itself

What the new version needs to do:
- Serve as a React application shell or web app platform
- Load documents, route between app variants, and manage application state
- Host the engine through a stable Rust-to-WASM API
- Handle progressive loading, error states, language packs, and app selection
- Provide distinct entry pages for classic, graphing, 3D, CAS, scientific, geometry, suite, notes, and board modes

Migration output:
- A new web app package with modern routing and state management
- App variant routing table
- Browser bootstrap page and config loader

#### source/web/web-common
Current role:
- Shared web UI helpers and JS export layer

New role:
- Become shared browser-facing utilities for the new frontend

What the new version needs to do:
- Expose browser-callable API methods
- Wrap document import/export, file handling, clipboard, and embedded mode interactions
- Provide common formatting and localization helpers

Migration output:
- Web-specific utility package
- Browser API layer with a documented contract

#### source/web/web-dev
Current role:
- Web developer helpers and runtime utilities

New role:
- Become development-only support code for the new web app

What the new version needs to do:
- Provide dev helpers, test fixtures, and browser-only utilities
- Stay out of production bundles unless absolutely required

Migration output:
- Dev-only package with strict bundling rules

#### source/web/editor-web
Current role:
- Browser formula editor implementation

New role:
- Become the web expression editor UI and interaction layer

What the new version needs to do:
- Render the editor using React or another modern UI layer
- Bind to the shared editor AST and commands
- Support formula input, caret handling, suggestions, and rendering feedback

Migration output:
- Web editor component library
- Editor state machine and command adapter

#### source/web/keyboard-web
Current role:
- Browser virtual keyboard UI

New role:
- Become a reusable keyboard component for all browser-based app variants

What the new version needs to do:
- Render keyboard layouts from the shared schema
- Support touch, mouse, and accessibility interactions
- Integrate with the editor and graphing input fields

Migration output:
- Keyboard UI component
- Themeable key renderer

#### source/web/renderer-web
Current role:
- Browser formula renderer

New role:
- Become the web renderer implementation for text, formulas, and graph annotations

What the new version needs to do:
- Render formulas on Canvas, SVG, or DOM as appropriate
- Support measuring, caching, and hit testing
- Support math text, rich labels, and symbolic output

Migration output:
- Web renderer package
- Performance benchmarks for complex formulas

#### source/web/canvas-web
Current role:
- Canvas implementation for the web stack

New role:
- Become the browser drawing backend for the shared canvas contract

What the new version needs to do:
- Map engine drawing commands to Canvas 2D, WebGL, or SVG primitives
- Provide consistent output across browsers

Migration output:
- Web canvas backend package

#### source/web/carota-web
Current role:
- Rich text editing component for the browser

New role:
- Replace with a modern text editing solution or keep only if there is a clear feature gap

What the new version needs to do:
- If kept, provide rich text support for notes and annotations
- If replaced, move to a more modern editor stack with better maintenance and accessibility

Migration output:
- Decide early whether this stays or is replaced

#### source/web/gwt-generator
Current role:
- GWT generation helper

New role:
- Replace with build-time code generation in the modern toolchain

What the new version needs to do:
- Generate browser bindings, schema clients, localization data, or type-safe resources
- Avoid relying on old GWT generation patterns

Migration output:
- Modern codegen package using the new build system

#### source/web/gwtutil
Current role:
- GWT utility and browser bridge code

New role:
- Become browser utilities and platform adapters in TypeScript or Rust/WASM bindings

What the new version needs to do:
- Provide DOM helpers, event wrappers, and browser integration utilities
- Keep platform-specific code small and obvious

Migration output:
- Replace with shared frontend utilities and typed browser wrappers

#### source/web/uitest
Current role:
- Cypress end-to-end tests

New role:
- Become the browser end-to-end verification suite for the new stack

What the new version needs to do:
- Exercise critical flows like launching app variants, entering formulas, drawing objects, saving documents, and restoring sessions
- Validate accessibility and responsive layouts

Migration output:
- New E2E test harness for the modern web app

### 4.3 Desktop Layer

#### source/desktop/desktop
Current role:
- Main desktop app orchestrator
- JVM application startup
- Platform-specific application runtime

New role:
- Either become a compatibility shell for the legacy app or be replaced by a new desktop frontend that uses the same engine as the browser and mobile apps

What the new version needs to do:
- Launch documents, select app variants, and render the UI
- Integrate with native file dialogs, printing, clipboard, and OS shortcuts
- Use the same engine protocol as web and mobile

Migration output:
- Modern desktop shell using the same engine contract

#### source/desktop/canvas-desktop
Current role:
- Desktop canvas backend

New role:
- Become the desktop drawing backend for the shared rendering contract

What the new version needs to do:
- Render via Skia, native canvas, or the desktop framework’s drawing API
- Match browser output as closely as possible

Migration output:
- Desktop rendering backend package

#### source/desktop/editor-desktop
Current role:
- Desktop math editor implementation

New role:
- Become the desktop editor UI component

What the new version needs to do:
- Share the same editor command model as the web version
- Integrate with desktop text input, menus, and shortcuts

Migration output:
- Desktop editor package

#### source/desktop/renderer-desktop
Current role:
- Desktop formula renderer

New role:
- Become the desktop formula and label renderer

What the new version needs to do:
- Provide identical output semantics to the web renderer where possible
- Support font fallback, measuring, and export-quality rendering

Migration output:
- Desktop renderer package

#### source/desktop/jogl2
Current role:
- JOGL OpenGL support for desktop 3D

New role:
- This should be removed if the new architecture does not require JOGL, or replaced by a modern graphics path if 3D remains a first-class desktop requirement

What the new version needs to do:
- Only exist if the desktop target really needs native OpenGL today
- Otherwise remove the dependency burden and simplify the runtime

Migration output:
- Prefer removal in the first modern architecture unless there is a hard 3D requirement

#### source/desktop/uitest if introduced later
Current role:
- Not present as a top-level package today, but should exist in the new architecture if desktop validation matters

New role:
- Add desktop end-to-end tests for launch, file open/save, tool interaction, and 3D rendering if the product needs it

Migration output:
- New desktop test harness

## 5. New Architecture To Build

The new repo should be split into these major layers.

### 5.1 Engine Layer

This is the heart of the rewrite.

Responsibilities:
- Parse expressions and commands
- Maintain mathematical objects and dependency graphs
- Evaluate algebraic, geometric, and numeric transformations
- Handle CAS-style symbolic operations
- Serialize and deserialize documents
- Record undo/redo history
- Apply localization-independent object semantics
- Remain deterministic and fully testable without a UI

Implementation guidance:
- Use Rust for the core engine
- Keep all state mutations explicit
- Do not let rendering concepts leak into the kernel
- Expose every operation through a documented API

### 5.2 Document Model Layer

This layer defines the project file format and runtime state model.

Responsibilities:
- Store objects, views, tools, algebra state, construction order, and metadata
- Support versioned migrations of saved documents
- Allow import/export for legacy GeoGebra files if required
- Preserve reproducibility of old files

Implementation guidance:
- Use versioned schemas
- Add compatibility tests with real files from the old repo
- Maintain a strict mapping between document state and engine state

### 5.3 Rendering Layer

Responsibilities:
- Convert engine state into visible graphics
- Support 2D geometry, algebra views, tables, labels, annotations, and 3D when necessary
- Separate model evaluation from paint scheduling
- Allow multiple rendering backends

Implementation guidance:
- Build a platform-neutral scene description
- Use backend adapters for browser and desktop
- Keep layout and measurement deterministic

### 5.4 Editor Layer

Responsibilities:
- Parse user text into math expressions
- Provide live editing, syntax coloring, completions, and validation
- Support calculator input, command input, and equation editing

Implementation guidance:
- Treat the editor as a state machine
- Keep the grammar shared across platforms
- Make keyboard and editor commands language-agnostic

### 5.5 Keyboard Layer

Responsibilities:
- Drive math keyboard layouts
- Support touch and accessibility input
- Provide consistent behavior across browser and desktop

Implementation guidance:
- Treat keyboard layouts as data
- Separate input actions from visual key rendering

### 5.6 App Shell Layer

Responsibilities:
- Decide which app variant is loaded
- Load localization resources
- Manage document lifecycle
- Connect engine, editor, renderer, keyboard, and UI
- Handle menus, toolbars, panels, and preferences

Implementation guidance:
- Build a product shell that is small and platform-specific
- Do not put engine rules inside the shell

### 5.7 Platform Adapters

Responsibilities:
- Browser adapter for web
- Desktop adapter for native shell
- Mobile adapters for iOS and Android
- Optional collaborative or embedded adapters

Implementation guidance:
- Use one shared engine API and one shared document schema
- Keep platform code thin and replaceable

## 6. Recommended New Repository Layout

A modern replacement repo could be organized like this:

- engine/
- document/
- parser/
- renderer-core/
- renderer-web/
- renderer-desktop/
- editor-core/
- editor-web/
- keyboard-core/
- keyboard-web/
- app-shell-web/
- app-shell-desktop/
- app-shell-ios/
- app-shell-android/
- tests/
- fixtures/
- schemas/
- tooling/
- docs/

A Rust-first version could also use a workspace like:

- crates/engine
- crates/document
- crates/parser
- crates/cas
- crates/rendering
- crates/ffi
- apps/web
- apps/desktop
- apps/ios
- apps/android
- tests/golden
- tests/e2e

## 7. Migration Phases

This is the implementation order I recommend.

### Phase 0: Audit and Freeze the Legacy Behavior

Objective:
- Capture what the current product actually does before replacing anything

Deliverables:
- Feature inventory
- App variant list
- Document format inventory
- Golden image set
- Golden output set for algebra and CAS
- Test corpus from representative user flows

Concrete tasks:
- Record startup behavior for each app variant
- Capture current document serialization output
- Save expected results for common commands
- Identify browser-only features and desktop-only features
- Identify features that can be dropped or delayed

Exit criteria:
- You can say which behaviors must be preserved in v1 and which can be deferred

### Phase 1: Define the New Product Contract

Objective:
- Freeze the public interface of the new system before coding the engine

Deliverables:
- Engine API specification
- Document schema specification
- Event model specification
- Rendering scene model specification
- Editor command model specification
- Keyboard action model specification

Concrete tasks:
- Decide which operations are sync and which are async
- Decide how documents are loaded and saved
- Decide how selections, styles, and views are represented
- Decide how errors are reported

Exit criteria:
- Every platform can talk to the engine through the same contract

### Phase 2: Build the Engine Core

Objective:
- Port the mathematical behavior into a new deterministic engine

Deliverables:
- Rust engine crate
- Parser crate
- Algebra and geometry object model
- CAS bridge or native symbolic module
- Undo/redo manager
- Document serialization layer

Concrete tasks:
- Translate the smallest stable set of core objects first
- Implement expression parsing
- Implement dependency tracking
- Implement numeric and symbolic evaluation
- Implement command execution
- Implement document load/save

Validation:
- Unit tests for every operation
- Golden tests against the legacy repo
- Fuzz tests for parser and import handling

Exit criteria:
- The engine can reproduce core behavior without a UI

### Phase 3: Build the Browser Shell

Objective:
- Provide the first modern user-facing product

Deliverables:
- Web app shell in TypeScript + React
- Engine integration through WebAssembly
- Document open/save workflow
- Basic canvas and editor UI
- Language loading and app variant routing

Concrete tasks:
- Build the root app shell
- Load the engine as a versioned runtime dependency
- Render the first view hierarchy
- Hook up editor input and keyboard actions
- Add app selection and page routing

Validation:
- Browser smoke tests
- Cross-browser checks
- Performance checks for startup and interaction latency

Exit criteria:
- A user can open a document, edit it, and see the graph update

### Phase 4: Rebuild the Renderer and Views

Objective:
- Match the legacy visual behavior closely enough for daily use

Deliverables:
- 2D canvas/SVG renderer
- Algebra view
- Graphics view
- Table view if needed
- Formula rendering
- Tool and object overlays

Concrete tasks:
- Implement coordinate transforms
- Implement hit testing
- Implement label layout
- Implement text and formula measurement
- Implement zoom/pan/drag behavior

Validation:
- Golden screenshots
- Pixel-diff tests where stable
- Manual checks for complex constructions

Exit criteria:
- Standard constructions look and behave correctly in the browser

### Phase 5: Rebuild Editor and Keyboard

Objective:
- Replace the formula input and keyboard experience

Deliverables:
- Shared editor state machine
- Keyboard layout model
- Browser keyboard UI
- Accessibility support

Concrete tasks:
- Implement caret movement and selection behavior
- Implement autocomplete and command insertion
- Implement math symbol input
- Recreate scientific and calculator-specific layouts

Validation:
- Input behavior tests
- Keyboard interaction tests
- Accessibility checks

Exit criteria:
- The editor can replace the old input path for the supported features

### Phase 6: Recreate App Variants

Objective:
- Make each user-facing product variant work on the new shell

Deliverables:
- Classic
- Graphing
- 3D
- CAS
- Scientific
- Geometry
- Suite
- Notes
- Board

Concrete tasks:
- Define variant-specific feature flags
- Define variant-specific startup state
- Define toolbar and panel presets
- Define variant-specific translations and icons

Validation:
- One smoke test per variant
- Variant-specific golden screenshots
- Startup and routing tests

Exit criteria:
- Every required app variant can launch and perform its key workflows

### Phase 7: Add Desktop and Mobile Frontends

Objective:
- Extend the same engine to native platforms

Deliverables:
- Desktop shell
- iOS app
- Android app

Concrete tasks:
- Port the shell to the desktop target
- Define file picker, clipboard, printing, and OS integration
- Port the app state and rendering adapters
- Reuse the same engine API and document schema

Validation:
- Platform-specific tests
- File round-trip tests
- Rendering checks against browser behavior

Exit criteria:
- The same document opens correctly on all platforms

### Phase 8: Compatibility and Legacy Import

Objective:
- Preserve user data and reduce migration pain

Deliverables:
- Legacy file importer
- Compatibility layer for old commands and documents
- Version migration system

Concrete tasks:
- Import old GeoGebra files
- Map legacy commands to new engine operations
- Preserve document metadata and view settings
- Report unsupported items clearly

Validation:
- Real-world legacy documents
- Regression suite for imported constructions

Exit criteria:
- Old user content survives the migration path

### Phase 9: Remove Legacy Code Paths

Objective:
- Eliminate the old architecture once the new one is stable

Deliverables:
- Deprecation plan
- Feature parity checklist
- Cleanup PRs

Concrete tasks:
- Remove GWT-specific runtime assumptions
- Remove duplicated browser and desktop logic
- Remove unused native bridges
- Archive old build paths after the cutover

Exit criteria:
- New repo becomes the source of truth

## 8. Feature Parity Checklist

Before declaring the migration complete, verify these areas.

### Core math and algebra
- Expression parsing
- Simplification
- Equation solving
- Numeric evaluation
- CAS operations
- Object dependencies
- Undo and redo

### Geometry
- Points, lines, circles, polygons
- Constraints and dynamic updates
- Coordinate transformations
- Angles, lengths, and measurements
- Object visibility and styling

### Views
- Algebra view
- Graphics view
- 3D view if kept
- Table view if kept
- Notes and board surfaces if kept

### Input and editing
- Math keyboard
- Scientific keyboard
- Formula editor
- Touch and pointer behavior
- Accessibility

### Runtime behavior
- Localization
- Theme and look-and-feel
- Document save/load
- Drag and drop
- Clipboard and export
- Error handling and recovery

### Platform behavior
- Browser support
- Desktop support
- Mobile support
- Printing and export
- Offline operation if required

## 9. Testing Strategy

Testing must be built into the migration from day one.

### 9.1 Golden Tests

Create test fixtures from the current repo and compare the new stack against them.

Use golden tests for:
- Algebra output
- Geometry state
- CAS results
- Document serialization
- Screenshot rendering
- Toolbar layout
- App variant startup

### 9.2 Engine Unit Tests

Every engine feature must have direct tests.

Test categories:
- Parser correctness
- Object creation
- Dependency propagation
- Undo/redo
- Serialization round trips
- Error handling

### 9.3 Platform Smoke Tests

Each platform needs a minimal smoke suite.

Browser smoke tests:
- Load app variant
- Insert object
- Change a property
- Save and reload

Desktop smoke tests:
- Start app
- Open and save file
- Use keyboard and mouse

Mobile smoke tests:
- Start app
- Pinch zoom
- Use editor input
- Rotate device if supported

### 9.4 Compatibility Tests

Use a real corpus of old documents to protect migration behavior.

Test categories:
- Old file import
- Old command parsing
- View restoration
- Incorrect or partial document recovery

## 10. Build and Release Plan

### Build system

The new repo should have:
- A single workspace definition
- Clearly separated packages or crates
- Strict linting and formatting
- Reproducible CI builds
- Artifact versioning

### CI pipeline

At minimum, CI should run:
- Engine unit tests
- Parser tests
- Document round-trip tests
- Web build
- Browser smoke tests
- Screenshot diff tests
- Desktop build if included
- Mobile build if included

### Release plan

Release in stages:
- Internal alpha for engine and browser shell
- Feature-limited beta for selected users
- Compatibility beta for imported documents
- Gradual platform expansion

## 11. What To Keep And What To Replace

Keep:
- The mathematical behavior
- The app variant idea
- The shared document concept
- The editor/keyboard separation
- The idea of platform-specific renderers

Replace:
- GWT runtime architecture
- JavaScript bootstrap complexity
- Deep UI coupling to the engine
- Native library entanglement in the UI layer
- Build logic that assumes one giant shared Java codebase

## 12. Recommended Rewrite Sequence For This Repo

If the rewrite starts from this repository, do it in this order:

1. Freeze the behavior of common, editor-base, renderer-base, and keyboard-base.
2. Build the new Rust engine with a minimal compatibility surface.
3. Build a modern web shell that talks only to that engine.
4. Port formula rendering and keyboard input.
5. Port the main app variants one by one.
6. Add compatibility import for legacy files.
7. Add desktop and mobile shells.
8. Remove the old GWT and JVM-specific UI code only after parity is proven.

## 13. Practical Definition Of Done

The migration is not done when the new code compiles.

It is done when:
- A user can open the same document in the old and new apps and get the same mathematical result
- The supported app variants behave consistently
- The browser app is stable and maintainable
- The core engine is tested independently of the UI
- Platform adapters are thin and replaceable
- The codebase can evolve without reintroducing the old monolithic dependencies

## 14. Final Recommendation

Do not attempt to rewrite GeoGebra as a single big-bang conversion.

The correct plan is:
- Extract and freeze behavior
- Build a new engine
- Build a new browser shell
- Port the most valuable app variants first
- Add mobile and desktop later
- Retire the old stack only after the new one has proven correctness

If you follow that order, the rewrite becomes manageable instead of risky.
