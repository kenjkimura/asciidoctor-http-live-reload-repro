# asciidoctor-http-live-reload-repro

`asciidoctor-maven-plugin` の `asciidoctor:http` goal におけるブラウザ自動更新の問題を再現するためのプロジェクトです。

English version: [README.md](README.md)

## 概要

`asciidoctor:http` goal は、生成した HTML にブラウザ側の live reload スクリプトを注入します。このスクリプトは、対象 HTML に対する `HEAD` リクエストのレスポンスヘッダーを見て、HTML が更新されたかどうかを判定します。

問題のある実装では、`AsciidoctorHandler` が `HEAD` リクエストに対して `205 Reset Content` を返します。Netty はこのステータスを本文なしのレスポンスとして扱うため、実際の応答は次のようになります。

```http
HTTP/1.1 205 Reset Content
expires: 0
content-type: text/html
content-length: 0
```

HTML が再生成されても、`live.js` が参照するヘッダーに有効な差分が出ないため、ブラウザが自動リロードされません。

`AsciidoctorHandler` の `HEAD` 応答を `205 Reset Content` から `200 OK` に変更すると、live reload が期待どおり動作します。

## プロジェクト構成

```text
.
├── .mvn/wrapper/maven-wrapper.properties
├── README.md
├── README.ja.md
├── mvnw
├── mvnw.cmd
├── pom.xml
└── src/docs/asciidoc/
    ├── index.adoc
    └── index.ja.adoc
```

## 前提条件

- JDK 17 以上
- ブラウザ

## 公式版プラグインで再現する

公式版プラグインで確認する場合は、隔離した Maven ローカルリポジトリを使ってサーバーを起動します。

```shell
./mvnw "-Dmaven.repo.local=target/maven-central-repo" "-Dasciidoctor.maven.plugin.version=3.2.0" asciidoctor:http
```

Windows PowerShell:

```shell
.\mvnw.cmd "-Dmaven.repo.local=target/maven-central-repo" "-Dasciidoctor.maven.plugin.version=3.2.0" asciidoctor:http
```

通常のローカル Maven リポジトリに修正版を install 済みの場合でも公式版の挙動を確認できるように、公式版の確認では隔離したリポジトリを使います。PowerShell では、上記のように `-D...` 引数を引用符で囲んでください。

Maven が出力した URL をブラウザで開きます。通常は次の URL です。

```text
http://localhost:2000/index
```

別のターミナルで `HEAD` 応答を確認します。

```shell
curl -I http://localhost:2000/index
```

Windows PowerShell:

```shell
curl.exe -I http://localhost:2000/index
```

問題が再現している場合、応答は次の特徴を持ちます。

- ステータスが `205 Reset Content`
- `content-length` が `0`
- `Last-Modified` が返らない
- `ETag` が返らない

次に [src/docs/asciidoc/index.adoc](src/docs/asciidoc/index.adoc) を編集します。たとえば次の行を変更します。

```adoc
Reload marker: initial version
```

保存して Maven が HTML を再生成するのを待ちます。問題のある実装では、ブラウザは自動更新されず、手動でリロードするまで古い表示のままになります。

日本語版の再現用 AsciiDoc は [src/docs/asciidoc/index.ja.adoc](src/docs/asciidoc/index.ja.adoc) です。内部レビューで説明を確認するときに使えます。

## 修正版プラグインで確認する

修正版は次の fork とブランチにあります。

- Fork: https://github.com/kenjkimura/asciidoctor-maven-plugin.git
- Branch: `fix/live-reload-head-response`

修正版プラグインを通常のローカル Maven リポジトリに install します。

```shell
git clone --branch fix/live-reload-head-response https://github.com/kenjkimura/asciidoctor-maven-plugin.git
cd asciidoctor-maven-plugin
mvn -pl asciidoctor-maven-plugin -am -DskipTests install
```

この再現プロジェクトに戻り、`asciidoctor:http` を起動します。

```shell
./mvnw asciidoctor:http
```

Windows PowerShell:

```shell
.\mvnw.cmd asciidoctor:http
```

Maven は通常のローカル Maven リポジトリから修正版の成果物を解決します。

ブラウザでページを開き、別ターミナルで `HEAD` 応答を確認します。

```shell
curl -I http://localhost:2000/index
```

Windows PowerShell:

```shell
curl.exe -I http://localhost:2000/index
```

修正版では、次のような応答になることを確認します。

- ステータスが `200 OK`
- `content-length` が生成された HTML のサイズになる
- live reload スクリプトが比較可能なメタデータを返せる

再度 [src/docs/asciidoc/index.adoc](src/docs/asciidoc/index.adoc) を編集して保存します。ブラウザが自動リロードされ、変更後の marker が手動更新なしで表示されることを確認します。

修正版を install したあとに公式版の挙動へ戻して確認したい場合は、上記の隔離 Maven リポジトリを使う公式版確認コマンドを実行するか、通常のローカル Maven リポジトリから修正版成果物を削除してください。

## サーバーの停止

Maven プロセスで `exit` または `quit` を入力するか、`Ctrl+C` を押します。

## Issue 報告用メモ

このリポジトリは、live reload の失敗が AsciiDoc 変換ではなく `HEAD` 応答のメタデータに起因することを示すためのものです。生成された HTML はディスク上で更新されますが、`HEAD` 応答が常に `205 Reset Content` かつ `content-length: 0` になるため、ブラウザ側の live reload チェックが変更を検知できません。