# asciidoctor-http-live-reload-repro

Reproduction project for an `asciidoctor-maven-plugin` `asciidoctor:http` live reload issue.

## Summary

The `asciidoctor:http` goal injects a browser-side live reload script into generated HTML. The script checks whether the served HTML has changed by polling response headers from `HEAD` requests.

In the affected implementation, `AsciidoctorHandler` returns `205 Reset Content` for `HEAD` requests. Netty treats this status as a response without content, so the effective response becomes similar to:

```http
HTTP/1.1 205 Reset Content
expires: 0
content-type: text/html
content-length: 0
```

When the HTML is regenerated, the headers observed by `live.js` do not change enough to detect the update, so the browser does not reload automatically.

Changing the `HEAD` response from `205 Reset Content` to `200 OK` in `AsciidoctorHandler` makes live reload work as expected.

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

## Reproduce with the Released Plugin

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

In another terminal, inspect the `HEAD` response:

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

Then edit [src/docs/asciidoc/index.adoc](src/docs/asciidoc/index.adoc), for example change this line:

```adoc
Reload marker: initial version
```

Save the file and wait for Maven to regenerate the HTML. The browser is expected to stay on the old content until it is manually reloaded.

## Verify with a Patched Plugin

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

Open the served page again and inspect the `HEAD` response:

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
- Other relevant metadata can be returned and compared by the live reload script

Edit and save [src/docs/asciidoc/index.adoc](src/docs/asciidoc/index.adoc) again. The browser should reload automatically and show the changed marker without manual refresh.

If you need to switch back to the released plugin after installing the fork, run the released-plugin check with the isolated repository command shown above, or remove the patched artifact from your normal local Maven repository.

## Stop the Server

Type `exit` or `quit` in the Maven process, or press `Ctrl+C`.

## Notes for Issue Report

This repository is intended to demonstrate that the live reload failure is caused by the `HEAD` response metadata, not by AsciiDoc conversion itself. The generated HTML is updated on disk, but the browser-side live reload check cannot detect the change when `HEAD` always resolves to `205 Reset Content` with `content-length: 0`.
