# vite4j

[English](README.md)

[Vite](https://ja.vite.dev) のエントリに必要なタグを、JVM のバックエンドから生成します。

Vite は自前の `index.html` ならタグを書き換えますが、バックエンドが出す HTML は Vite からは見えません。代わりに「どのエントリに何が必要か」をマニフェストに記録します。vite4j はそれを読みます。

ファイル名を手で書いても、バンドラがチャンクを分割するまでは動きます。分割された瞬間、切り出された CSS はビルドされ、配信され、どこからもリンクされないまま一度も適用されなくなります。

## インストール

```xml
<dependency>
  <groupId>io.github.katayama8000</groupId>
  <artifactId>vite4j</artifactId>
  <version>0.1.0</version>
</dependency>
```

Java 11 以降。Jackson は optional で、使うのは `JacksonManifestParser` だけです。

## 使い方

```js
// vite.config.js
export default defineConfig({
  build: { manifest: true, rollupOptions: { input: "src/main.tsx" } },
})
```

起動時に 1 回読みます。

```java
ViteManifest manifest;
try (InputStream in = loader.getResourceAsStream("public/.vite/manifest.json")) {
    manifest = ViteManifest.of(new JacksonManifestParser().parse(in));
}

ViteAssets assets = ViteAssets.builder(manifest)
        .assetUrl(path -> "/assets/" + path)
        .build();
```

ページごとに生成します。

```java
assets.html("src/main.tsx");
```

```html
<link rel="stylesheet" href="/assets/assets/shared-ChJ_j-JJ.css">
<link rel="stylesheet" href="/assets/assets/main-5UjPuW-k.css">
<script type="module" src="/assets/assets/main-BRBmoGS9.js"></script>
```

`assetUrl` にはマニフェストに書かれたままのパスが渡ります。CDN のホスト、デプロイのバージョン、ダイジェスト付きディレクトリはここに入れてください。

## オプション

| | |
| --- | --- |
| `assetUrl(fn)` | マニフェストのパスを URL にする。既定は `/` を前置 |
| `modulePreload(true)` | import したチャンクごとに `<link rel="modulepreload">` を出す。既定は off |

## 開発時

dev server が動いている間はマニフェストがありません。`ViteDevServer` が本番用と同じ `ViteTags` インタフェースを実装しているので、テンプレート側は分岐せずに済みます。

```java
ViteTags vite = devMode
        ? ViteDevServer.at("http://localhost:5173/").withReactRefresh()
        : ViteAssets.builder(manifest).assetUrl(assetUrl).build();
```

`withReactRefresh()` は `@vitejs/plugin-react` が要求するプリアンブルを足します。

## 別の JSON ライブラリを使う

`ManifestParser` はメソッド 1 つです。

```java
ManifestParser parser = json -> { /* → Map<String, Chunk> */ };
```

## マニフェストの読み方

[Backend Integration](https://ja.vite.dev/guide/backend-integration) の手順に従います。見落としやすいのは、**各チャンクが自分の CSS しか列挙しない**ことです。import 先のチャンクもたどる必要があります。`manifest[entry].css` を読んで終わりにすると、バンドラが切り出した分が漏れます。

順序は import 側が先、自分の CSS が最後です。モジュールの実行順であり、Vite が出力する順序でもあります。共有 CSS は 1 回だけリンクされ、循環しても停止し、マニフェストに無いチャンクは例外を投げずにスキップします。

## ライセンス

Apache-2.0
