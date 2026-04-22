# Restart Guide

This repo is a Gradle/GWT project. In GitHub Codespaces, the most reliable way to bring the web UI back is to run a plain static server for the generated web assets on port 8888.

## Prerequisites

- Java 17
- The repository checked out at the project root
- A terminal opened in the repo root

If Java 17 is not available yet, install it first:

```bash
sudo apt-get update
sudo apt-get install -y openjdk-17-jdk
```

## Fastest way to run the web UI

This is the simplest restart path after reopening the Codespace:

```bash
export JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64
export PATH="$JAVA_HOME/bin:$PATH"
python3 -m http.server 8888 --directory source/web/web/war
```

Then forward port 8888 and open:

```text
https://<your-codespace>-8888.app.github.dev/classic.html?prerelease=false
```

## If the HTML files are missing or stale

Regenerate the web HTML pages first:

```bash
export JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64
export PATH="$JAVA_HOME/bin:$PATH"
./gradlew :web:web:copyHtml
```

If you want the files to point at the stable GeoGebra runtime CDN, keep the generated HTML patched so the browser does not need the local Super Dev code server.

## Original GeoGebra dev mode

The repo also has a GWT development mode task:

```bash
export JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64
export PATH="$JAVA_HOME/bin:$PATH"
xvfb-run -a ./gradlew :web:web:run
```

Important notes:

- This starts the local code server on port 9876 and a dev server on port 8888.
- In Codespaces this mode is fragile and may fail because it needs a GUI/X11-like environment.
- If the browser shows a message about Super Dev Mode on port 9876, use the static-server path above instead.

## Useful checks

```bash
java -version
netstat -tulnp 2>/dev/null | grep -E ":(8888|9876)\\b" || true
```

## Stop the server

If the plain HTTP server is running in a terminal, stop it with Ctrl+C.