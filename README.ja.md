# vite-manifest

[English](README.md)

[Vite](https://ja.vite.dev) がビルド時に書き出すマニフェストを JVM のバックエンドから読み、ページに必要なタグを生成するライブラリです。

Vite は自前の `index.html` なら中のタグを書き換えてくれます。しかし Play や Spring、Micronaut、http4s などがレンダリングする HTML は Vite からは見えないので、書き換えようがありません。代わりに Vite は「どのエントリに何が必要か」をマニフェストに記録し、タグの生成はバックエンドに委ねます。このライブラリはその記録を読みます。

ファイル名を決め打ちしても、しばらくは動いてしまいます。rollup はエントリが静的に import したものをすべてエントリチャンクに残していたので、CSS は 1 本しか出ず、名前を直接書いても問題ありませんでした。ところが rolldown は、動的 import されるチャンクと共有しているモジュールを別チャンクに切り出します。CSS もそれについて出ていき、どこからもリンクされないファイルになります。ビルドもされ、配信もされ、しかし一度も適用されない CSS ができあがります。

同種の統合は [Laravel](https://github.com/laravel/vite-plugin)、[Rails](https://github.com/ElMassimo/vite_ruby)、[Django](https://github.com/MrBin99/django-vite)、[Symfony](https://github.com/lhapaipai/vite-bundle)、Go、Rust にはありますが、JVM にはありませんでした。

## インストール

```xml
<dependency>
  <groupId>dev.vitemanifest</groupId>
  <artifactId>vite-manifest-core</artifactId>
  <version>0.1.0</version>
</dependency>
<!-- 任意: すでに使っている Jackson でマニフェストをパースする -->
<dependency>
  <groupId>dev.vitemanifest</groupId>
  <artifactId>vite-manifest-jackson</artifactId>
  <version>0.1.0</version>
</dependency>
```

`vite-manifest-core` は依存ゼロです。`vite-manifest-jackson` は Jackson を `provided` で宣言しているので、バージョンの主導権はアプリケーション側にあります。

## 使い方

まず Vite にマニフェストを書かせます。

```js
// vite.config.js
export default defineConfig({
  build: {
    manifest: true,
    rollupOptions: { input: "src/main.tsx" },
  },
})
```

起動時に一度読み、リクエストごとにタグを生成します。

```java
ManifestParser parser = new JacksonManifestParser();
ViteManifest manifest;
try (InputStream in = classLoader.getResourceAsStream("public/.vite/manifest.json")) {
    manifest = ViteManifest.of(parser.parse(in));
}

ViteAssets assets = ViteAssets.builder(manifest)
        .assetUrl(path -> "/assets/" + path)
        .modulePreload(true)
        .build();

assets.html("src/main.tsx");
```

```html
<link rel="stylesheet" href="/assets/assets/shared-ChJ_j-JJ.css">
<link rel="stylesheet" href="/assets/assets/main-5UjPuW-k.css">
<link rel="modulepreload" href="/assets/assets/shared-B7PI925R.js">
<script type="module" src="/assets/assets/main-BRBmoGS9.js"></script>
```

`assetUrl` は、CDN のホスト名、デプロイごとのバージョン、ダイジェスト付きのディレクトリなどを差し込む場所です。マニフェストに書かれたままのパスが渡ってきます。

### 開発時

dev server が動いている間はマニフェストもビルド済みファイルもありません。`ViteDevServer` がそのモードのタグを生成します。本番用の `ViteAssets` と同じ `ViteTags` インタフェースを実装しているので、テンプレート側はどちらのモードか気にせず同じ書き方で済みます。

```java
ViteTags vite = devMode
        ? ViteDevServer.at("http://localhost:5173/").withReactRefresh()
        : ViteAssets.builder(manifest).assetUrl(assetUrl).build();

vite.html("src/main.tsx");
```

`withReactRefresh()` は、`@vitejs/plugin-react` がコンポーネントの読み込み前に要求するプリアンブルを追加します。

### Jackson を使わない場合

`ManifestParser` はメソッド 1 つだけのインタフェースなので、どの JSON ライブラリでも実装できます。

```java
ManifestParser parser = json -> {
    Map<String, Chunk> chunks = new LinkedHashMap<>();
    // ... file, css, imports を詰める
    return chunks;
};
```

## マニフェストをどう解釈するか

Vite の [Backend Integration](https://ja.vite.dev/guide/backend-integration) ガイドにある 4 つの手順に従います。見落としやすいのは 2 番目です。**各チャンクは自分の CSS しか列挙しません。** つまり import しているチャンクもたどる必要があります。`manifest[entry].css` を読んで終わりにすると、バンドラが切り出した分がまるごと漏れます。

順序は import 側が先、そのチャンク自身の CSS が最後です。これはモジュールの実行順であり、Vite 自身が出力する順序でもあります。したがって、同じ詳細度で競合したときにどちらのルールが勝つかは、Vite がレンダリングしたページと同じになります。

2 つのチャンクが共有する CSS は 1 回だけリンクされます。循環があっても停止します。マニフェストに存在しないチャンクは例外を投げずにスキップします。マニフェストはビルドの成果物であり、スタイルが一部欠けたページのほうが、レンダリングに失敗するページよりましだからです。

## 要件

Java 11 以降。

## ライセンス

Apache-2.0
