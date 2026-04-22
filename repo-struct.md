# GeoGebra Repo Structure

This repository is the GeoGebra math apps codebase. It is a Gradle composite build with shared core modules plus separate web and desktop targets.

## Top-Level Layout

- `settings.gradle.kts` includes the three major builds: shared, desktop, and web.
- `source/shared` contains the shared math and UI infrastructure used by multiple platforms.
- `source/web` contains the browser app, GWT modules, HTML templates, generated assets, and web-specific UI code.
- `source/desktop` contains the desktop JVM app and desktop-native dependencies.
- `source/build-logic` contains Gradle convention plugins and build helpers.

## How the Web App Is Built

The web app is not a Node/React frontend. It is a Java-to-JavaScript app built with GWT.

The main web build file is:

- `source/web/web/build.gradle.kts`

Key things it does:

- Declares the GWT modules to compile.
- Generates CSS from Sass into `source/web/web/war/css`.
- Generates HTML entry pages such as `classic.html`, `graphing.html`, `cas.html`, and `suite.html`.
- Defines the dev-mode `run` task.

## What Runs on Port 8888

Port 8888 serves the web output directory under `source/web/web/war`.

That directory contains:

- HTML entry pages such as `classic.html` and `graphing.html`
- CSS under `war/css`
- GWT output and helper assets such as `web3d/`, `webSimple/`, `dev/`, and icons/fonts

In the normal browser flow:

1. The browser opens an HTML page from `war`.
2. The HTML template injects a `module.nocache.js` script.
3. That script loads the compiled Java-to-JS application.
4. The Java app creates the UI and renders it into the page.

The HTML template is here:

- `source/web/web/src/main/resources/org/geogebra/web/resources/html/app-template.html`

The Java startup path is here:

- `source/web/web/src/main/java/org/geogebra/web/full/Web.java`
- `source/web/web/src/main/java/org/geogebra/web/full2d/Web2D.java`
- `source/web/web/src/main/java/org/geogebra/web/geogebra3D/Web3D.java`

## Java Startup Flow

The core browser entry point is a GWT `EntryPoint` class.

- `Web.gwt.xml` declares the 2D web entry point class.
- `Web3D.gwt.xml` declares the 3D web entry point class.
- `Editor.gwt.xml` declares the editor module entry point.

Important classes:

- `source/web/web/src/main/java/org/geogebra/web/full/Web.java`
- `source/web/web/src/main/java/org/geogebra/web/full2d/Web2D.java`
- `source/web/web/src/main/java/org/geogebra/web/geogebra3D/Web3D.java`
- `source/web/web/src/main/java/org/geogebra/web/editor/EditorEntry.java`

The high-level flow is:

1. The HTML page sets bootstrap variables such as `codebase`, `module`, `prerelease`, and `appID`.
2. It injects the GWT bootstrap script.
3. GWT calls the Java `onModuleLoad()` method.
4. `Web.onModuleLoad()` initializes the app and calls into the app frame/UI layer.
5. The actual GeoGebra views, toolbars, algebra view, graphics view, and dialogs are then rendered by the Java UI layer compiled to browser code.

## HTML, CSS, and Assets

Yes, this project uses HTML and CSS heavily.

- HTML templates live in `source/web/web/src/main/resources/org/geogebra/web/resources/html`
- Sass sources live in `source/web/web/src/main/resources/scss`
- Generated CSS goes to `source/web/web/war/css`
- Icons, fonts, JS helpers, and other assets live under `source/web/web/src/main/resources/org/geogebra/web/resources`

The HTML generation code is in:

- `source/web/web/build.gradle.kts`

## App Variants

This repo builds multiple GeoGebra app variants from the same codebase.

Defined variants include:

- Classic
- Graphing Calculator
- 3D Graphing Calculator
- CAS Calculator
- Scientific Calculator
- Geometry
- Calculator Suite
- Notes
- Board

Those variants are defined in:

- `source/build-logic/convention/src/main/kotlin/app-specs-convention.gradle.kts`

## Why The Browser Sometimes Asked For Port 9876

That happened because the dev bootstrap script for `web3d` was using Super Dev Mode.

The file was:

- `source/web/web/war/web3d/web3d.nocache.js`

That script tried to load from a code server on port 9876. In Codespaces, that can fail or behave inconsistently. The stable workaround is to serve the generated `war` folder statically on 8888.

## Frontend vs Backend

This repo is mostly the frontend application plus build tooling.

- The browser UI is produced from Java, GWT, HTML templates, and CSS.
- Some external services are contacted at runtime, such as GeoGebra API endpoints and asset/CDN URLs.
- The local 8888 server is not a separate Node backend; it is just serving the generated web output.

## Useful Files To Read First

- `README.md`
- `source/web/web/build.gradle.kts`
- `source/web/web/src/main/resources/org/geogebra/web/resources/html/app-template.html`
- `source/web/web/src/main/java/org/geogebra/web/full/Web.java`
- `source/web/web/src/main/java/org/geogebra/web/Web.gwt.xml`
- `source/web/web/src/main/java/org/geogebra/web/Web3D.gwt.xml`
