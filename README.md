# asciidoctor-http-live-reload-repro

Reproduction project for an `asciidoctor-maven-plugin` `asciidoctor:http` live reload issue.

## Summary

With `asciidoctor:http` in `asciidoctor-maven-plugin` 3.2.0, saving an AsciiDoc source file leaves the browser page showing stale content.

With the fixed plugin, the browser page updates after the source file is saved.

## Prerequisites

- JDK 17 or later
- A browser

## Reproduce the Browser Reload Failure

Steps:

1. Start `asciidoctor:http` with `asciidoctor-maven-plugin` 3.2.0:

	```shell
	./mvnw "-Dmaven.repo.local=target/maven-central-repo" "-Dasciidoctor.maven.plugin.version=3.2.0" asciidoctor:http
	```

2. Open the URL printed by Maven, usually:

	```text
	http://localhost:2000/index
	```

3. Edit [src/docs/asciidoc/index.adoc](src/docs/asciidoc/index.adoc), save the file, and wait for Maven to process the change.

4. Check the browser page.

Expected result:

The generated HTML changes, but the browser page still shows the old content.

## Cause

The browser-side live reload script checks for changes by using the `HEAD` response.

With `asciidoctor-maven-plugin` 3.2.0, the `HEAD` response is `205 Reset Content` with `content-length: 0`, so the live reload script cannot detect that the generated HTML changed.

## Verify the Browser Reload Fix

With the fixed plugin, the `HEAD` response is `200 OK` and `content-length` is based on the generated HTML size, so the live reload script can detect the change.

Steps:

1. Use the patched fork and branch:

	- Fork: https://github.com/kenjkimura/asciidoctor-maven-plugin.git
	- Branch: `fix/live-reload-head-response`

2. Install the patched plugin into your normal local Maven repository:

	```shell
	git clone --branch fix/live-reload-head-response https://github.com/kenjkimura/asciidoctor-maven-plugin.git
	cd asciidoctor-maven-plugin
	mvn -pl asciidoctor-maven-plugin -am -DskipTests install
	```

3. Return to this reproduction project and start `asciidoctor:http`:

	```shell
	./mvnw asciidoctor:http
	```

	Maven resolves the patched artifact from your normal local Maven repository.

4. Open the served page again, then edit and save [src/docs/asciidoc/index.adoc](src/docs/asciidoc/index.adoc).

5. Check the browser page.

Expected result:

The browser page updates automatically to show the latest content.
