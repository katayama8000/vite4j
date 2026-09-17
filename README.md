# vite-manifest

[日本語](README.ja.md)

Read [Vite](https://vite.dev)'s build manifest from a JVM backend and render the tags a page needs.

Vite rewrites the tags in its own `index.html`. A page rendered by Play, Spring, Micronaut, http4s or
anything else is markup Vite never sees, so it writes down what each entry needs instead and leaves
the tags to you. This library reads that record.

Hardcoding the filenames works right up until it doesn't. Rollup kept everything an entry imported
statically inside the entry chunk, so one stylesheet came out and naming it by hand was fine.
Rolldown splits a module shared with a dynamically imported chunk into a chunk of its own, and its
CSS follows it into a file nothing links — the rules are built, served, and never applied.

There are integrations like this for [Laravel](https://github.com/laravel/vite-plugin),
[Rails](https://github.com/ElMassimo/vite_ruby), [Django](https://github.com/MrBin99/django-vite),
[Symfony](https://github.com/lhapaipai/vite-bundle), Go and Rust. There was none for the JVM.

## Install

```xml
<dependency>
  <groupId>dev.vitemanifest</groupId>
  <artifactId>vite-manifest-core</artifactId>
  <version>0.1.0</version>
</dependency>
<!-- optional: parses the manifest with the Jackson you already have -->
<dependency>
  <groupId>dev.vitemanifest</groupId>
  <artifactId>vite-manifest-jackson</artifactId>
  <version>0.1.0</version>
</dependency>
```

`vite-manifest-core` has no dependencies. `vite-manifest-jackson` declares Jackson as `provided`, so
your application keeps control of the version.

## Use

Tell Vite to write a manifest:

```js
// vite.config.js
export default defineConfig({
  build: {
    manifest: true,
    rollupOptions: { input: "src/main.tsx" },
  },
})
```

Read it once at startup and render tags per request:

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

`assetUrl` is where a CDN host, a deployment version or a digest directory goes. It receives the path
exactly as the manifest spells it.

### Development

While the dev server runs there is no manifest and no built file. `ViteDevServer` renders the tags for
that mode behind the same `ViteTags` interface, so a template holds one of the two and does not branch:

```java
ViteTags vite = devMode
        ? ViteDevServer.at("http://localhost:5173/").withReactRefresh()
        : ViteAssets.builder(manifest).assetUrl(assetUrl).build();

vite.html("src/main.tsx");
```

`withReactRefresh()` adds the preamble `@vitejs/plugin-react` requires before any component module runs.

### Without Jackson

`ManifestParser` is a single method, so any JSON library will do:

```java
ManifestParser parser = json -> {
    Map<String, Chunk> chunks = new LinkedHashMap<>();
    // ... file, css, imports
    return chunks;
};
```

## What it does with the manifest

It follows the four steps in Vite's [Backend Integration](https://vite.dev/guide/backend-integration.html)
guide. The one that is easy to miss is the second: **a chunk lists only its own stylesheets**, so the
chunks it imports have to be walked as well. Reading `manifest[entry].css` and stopping there misses
everything the bundler split out.

Order is imports first, the chunk's own stylesheets last. That is the order the modules run in and the
order Vite itself emits, so a rule wins here exactly as it would on a page Vite rendered. A stylesheet
two chunks share is linked once, cycles terminate, and a chunk the manifest does not describe is
skipped rather than throwing: the manifest is build output, and a page half-styled beats a page that
fails to render.

## Requirements

Java 11 or later.

## License

Apache-2.0
