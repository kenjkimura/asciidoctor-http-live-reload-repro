# asciidoctor-http-live-reload-repro

Reproduction project for an `asciidoctor-maven-plugin` `asciidoctor:http` live reload issue.

## Summary

The asciidoctor:http goal should reload the browser automatically when an AsciiDoc source file is saved and the HTML is regenerated.

This project demonstrates that version 3.2.0 regenerates the HTML file, but the browser does not reload automatically.

The browser-side live reload script detects changes by polling the served HTML with HEAD requests. However, AsciidoctorHandler returns 205 Reset Content with content-length: 0, preventing the script from detecting the regenerated HTML.

Changing the HEAD response status to 200 OK allows the browser to detect the change and reload automatically.

## Project Layout

```text
.
├── .mvn/wrapper/maven-wrapper.properties
├── README.md
├── mvnw
├── mvnw.cmd
├── pom.xml
└── src/docs/asciidoc/index.adoc
```

## Prerequisites

- JDK 17 or later
- A browser

## Reproduce the Browser Reload Failure

Start the server with the released plugin by using an isolated local Maven repository:

```shell
./mvnw "-Dmaven.repo.local=target/maven-central-repo" "-Dasciidoctor.maven.plugin.version=3.2.0" asciidoctor:http
```

Windows PowerShell:

```shell
.\mvnw.cmd "-Dmaven.repo.local=target/maven-central-repo" "-Dasciidoctor.maven.plugin.version=3.2.0" asciidoctor:http
```

The isolated repository keeps the released-plugin check independent from any patched artifact installed in your normal local Maven repository. In PowerShell, keep the `-D...` arguments quoted as shown above.

Open the URL printed by Maven, usually:

```text
http://localhost:2000/index
```

Then edit [src/docs/asciidoc/index.adoc](src/docs/asciidoc/index.adoc), for example change this line:

```adoc
Reload marker: initial version
```

Save the file and wait for Maven to regenerate the HTML.

Expected result with the released plugin: the generated HTML changes, but the browser does not reload automatically. If you reload the page manually, the updated content is shown.

You can also inspect the `HEAD` response that causes the browser-side live reload check to miss the update:

```shell
curl -I http://localhost:2000/index
```

Windows PowerShell:

```shell
curl.exe -I http://localhost:2000/index
```

Expected affected response characteristics:

- Status is `205 Reset Content`
- `content-length` is `0`
- `Last-Modified` is absent
- `ETag` is absent

## Verify the Browser Reload Fix

Use the patched fork and branch:

- Fork: https://github.com/kenjkimura/asciidoctor-maven-plugin.git
- Branch: `fix/live-reload-head-response`

Install the patched plugin into your normal local Maven repository:

```shell
git clone --branch fix/live-reload-head-response https://github.com/kenjkimura/asciidoctor-maven-plugin.git
cd asciidoctor-maven-plugin
mvn -pl asciidoctor-maven-plugin -am -DskipTests install
```

Return to this reproduction project and start `asciidoctor:http`:

```shell
./mvnw asciidoctor:http
```

Windows PowerShell:

```shell
.\mvnw.cmd asciidoctor:http
```

Maven resolves the patched artifact from your normal local repository.

Open the served page again. Edit and save [src/docs/asciidoc/index.adoc](src/docs/asciidoc/index.adoc) again.

Expected result with the patched plugin: the browser reloads automatically and shows the changed marker without manual refresh.

You can also inspect the `HEAD` response used by the live reload script:

```shell
curl -I http://localhost:2000/index
```

Windows PowerShell:

```shell
curl.exe -I http://localhost:2000/index
```

Expected patched response characteristics:

- Status is `200 OK`
- `content-length` matches the generated HTML size

If you need to switch back to the released plugin after installing the fork, run the released-plugin check with the isolated repository command shown above, or remove the patched artifact from your normal local Maven repository.

## Stop the Server

Type `exit` or `quit` in the Maven process, or press `Ctrl+C`.

## Notes for Issue Report

This repository is intended to demonstrate that the live reload failure is caused by the `HEAD` response metadata, not by AsciiDoc conversion itself. The generated HTML is updated on disk, but the browser-side live reload check cannot detect the change when `HEAD` always resolves to `205 Reset Content` with `content-length: 0`.
