---
lang: en-US
title: Migrate to V2
---

# Migrate to V2

There have been several breaking changes in the upgrade to v2. Most of these changes are to simplify the configuration and rely more on standards provided by Vite rather than using custom options.

[[toc]]

## MV3 HMR is here!

If you were using `vite build --watch` while developing your extension to reload on change, you can now use `vite dev` instead!

```json
  "scripts": {
    "dev": "vite build --watch --mode development", // [!code --]
    "dev": "vite dev", // [!code ++]
    "build": "vite build",
```

::: warning
For Chrome, you'll need to be on version 110, which was released Feb 13, 2023. Make sure you're browser is up to date, or HMR will not work.
:::

## Drop CJS support

`web-ext` was upgraded from v6.5 &rarr; v7.2. The big change here is that the library dropped CJS support.

In your `package.json`, it's now required to add `"type": "module"`.

```json
  "version": "1.2.5",
  "type": "module", // [!code ++]
  "scripts": {
```

If you have any config files like tailwind that were common JS, rename them to use the `.cjs` file extension.

## No more `assets` option

In v1, everything in the `assets` directory was copied to the output directory. That sound familiar? Well, that's exactly what the [`public` directory](https://vitejs.dev/guide/assets.html#the-public-directory) does.

If your config used to look like this:

```ts
export default defineConfig({
  root: path.resolve(__dirname, "src"),
  plugins: [
    webExtension({
      manifest: "manifest.json",
      assets: "assets",
    }),
  ],
});
```

To migrate, move all your assets into the `<viteRoot>/public` directory and delete the `assets` option from your config.

```bash
mkdir public
git mv assets/* public
```

```ts
plugins: [
  webExtension({
    manifest: "manifest.json",
    assets: "assets", // [!code --]
  }),
];
```

## No more `generated:` prefix

In v1, you prefixed a file in the input manifest with `generated:` to list it as required in the final `manifest.json`, but that it shouldn't be treated as a input that needs built. This was mostly used for CSS files generated by JS inputs.

In v2, the final `manifest.json` is filled out automatically based on your build output! So just remove any files prefixed with `generated:`, and it will magically work!

```json
    "content_scripts": [
        {
            "matches": ["<all_urls>"],
            "js": ["some-scripts.ts"],
            "css": ["generated:some-scripts.ts"],  // [!code --]
        }
    ]
```

## Default `manifest` option

By default, the `manifest` option in the plugin's configuration will be `"manifest.json"`, meaning you can remove that option entirly if you were using that as the filename.

```ts
webExtension({
  manifest: "manifest.json", // [!code --]
});
```

If you're using a different file to generate your manifest, no changes are required.

If you're using a function to generate your manifest, no changes are required. But you should consider adding any files responsible for generating your manifest to the `watchFilePaths` option. See the next section for more details.

## Restart browser when changing `manifest.json`

Whenever you change the file defined in your `manifest` option, the browser will automatically restart and reinstall your extension. This should help you iterate faster when updating permissions, content scripts, or any other change you might make.

If you're using a function to generate your manifest, to take advantage of this new behavior you'll need to manually add all files that generate your manifest to the `watchFilePaths` option.

## Browser startup config changes

To change browser startup behavior, you can still pass the [`webExtConfig` option](/config/plugin-options#webextconfig). But you can now also use automatically discovered config files without importing them in your `vite.config.ts`. See [Configure Browser Startup](/guide/configure-browser-startup.md) for more details.

<!-- prettier-ignore -->
```ts
import { defineConfig } from "vite";
import webExtension, { readJsonFile } from "vite-plugin-web-extension";

function loadWebExtConfig() { // [!code --]
  try { // [!code --]
    return require("./.web-ext.config.json"); // [!code --]
  } catch { // [!code --]
    return undefined; // [!code --]
  } // [!code --]
} // [!code --]

export default defineConfig({
  plugins: [
    webExtension({
      manifest: "src/mainfest.json",
      webExtConfig: loadWebExtConfig(), // [!code --]
    }),
  ],
});
```

## Build process changes

In v1, the main build step bunbled your HTML entrypoints, and follow-up build steps built your content scripts, background page, and other entrypoints.

In v2, the main build step now just writes the `manifest.json` to the output directory. HTML entrypoints, content scripts, the background script, and additional entrypoints are all built in "children" steps.

> The new build output and logging clearly states what files are being built when, so check that out if you're curious to learn more.

In watch and dev mode, the child build processes now happen in series instead of in parallel.

All these changes have several effects on builds:

- The **generated files can be automatically added to the final manifest** (see [No More `generated:` Prefix](#no-more-generated-prefix))
- **Less memory is consumed** because it only holds the manifest and a single in-progress build in memory at one time
- **No more multiple reloads** during a single rebuild in watch mode!
- **Build extensions without a UI**, you no longer have to specify a `dummy.html` when your manifest doesn't include an HTML entrypoint

## Summary of removed options

- `assets`: See [No More `assets` Option](#no-more-assets-option)
- `writeAssetsTo`: Without an `assets` option, this does nothing
- `writeManifestTo`: To change where the extension builds to, use [Vite's `build.outDir`](https://vitejs.dev/config/build-options.html#build-outdir) option instead
- `verbose`: Use the `--debug` flag when performing a build (`vite build --debug`) to see more output
- `serviceWorkerType`: Deprecated field has been removed. As of [v1.0.3](https://github.com/aklinker1/vite-plugin-web-extension/releases/tag/v1.0.3), this option has had no effect, so you can just remove it