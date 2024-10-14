---
title: Creating a React component library with Vite
description: Using Vite's library mode to build a React component library
slug: react-component-library-with-vite
date: 2024-10-08 00:00:00+0000
image: cover.jpg
categories:
  - React
  - Vite
tags:
  - React
  - Vite
  - Rollup
  - Storybook
  - Vitest
weight: 1
links:
  - title: How to Create and Publish a React Component Library
    description: A tutorial for creating a React component library with Rollup
    website: https://dev.to/alexeagleson/how-to-create-and-publish-a-react-component-library-2oe
  - title: Build a React component library with TypeScript and Vite
    description: A tutorial for creating a React component library with Vite
    website: https://victorlillo.dev/blog/react-typescript-vite-component-librarymenu
---

## Context

A brief discussion with a client recently reminded me of something I'd long wanted to look into. The conversation was around the consolidation of designs across their rather broad suite of applications, and given the frequency with which React was used it got me wondering if it might be worth looking into creating a reusable component library.

Plenty of resources popped up when I started looking into the topic, and ultimately I discovered two promising avenues:

- Rollup
- Vite (or more specifically, Vite's [Library Mode](https://vite.dev/guide/build.html#library-mode))

Of the above options, Rollup seemed to be the more "traditional" approach and so that's where I started.

## Rollup

[This post](https://dev.to/alexeagleson/how-to-create-and-publish-a-react-component-library-2oe) by [Alex Eagleson](https://dev.to/alexeagleson) was instrumental in helping me get my head around what needed to be done. There's also a fantastic accompanying video tutorial linked on his post.

The tutorial is a little outdated so not all of the steps work as described, however there is a good amount of discussion in the comments with updated instructions. In any case, it proved to be informative enough to get me started in building a component library.

Hopefully it saves any pain, but one thing that caught me was a change to `@rollup/plugin-typescript` in version 12 which was causing errors, especially when trying to create multiple outputs. Rolling back to version 11 restored the original functionality and allowed me to progress with the tutorial.

This resource was incredibly helpful in introducing me to the core concepts of how to bundle and publish a component library, and I highly recommend reading through it for anyone unfamiliar with Rollup and how to use it in this context.

## Vite

After running through the tutorial above and using the resulting library's simple button component in another React application, I decided to look at Vite before continuing.

Of the resources I used, [this post](https://victorlillo.dev/blog/react-typescript-vite-component-library) by [Víctor Lillo](https://victorlillo.dev/) proved to be the most complete as it covered all of the aspects I wanted to look at.

Initially I disliked this approach due to the need to create a full-blown React application and subsequently remove all the bits we didn't need, however it was still _relatively_ painless and came with a few nice things working straight out of the box &mdash; CSS as an example.

### Exporting components

I did prefer the way components were exported in the first tutorial I followed, so I stuck with that here. This approach used an explicit `index.ts` file at each level of the hierarchy.

`src/components/Button/index.ts`

```ts
export { default } from "./Button";
```

`src/components/index.ts`

```ts
export { default as Button } from "./Button";
```

`src/index.ts`

```ts
export * from "./components";
```

### Library mode

The initial config required on the Vite side was relatively straightforward. Setting the required values for `build.lib` under `vite.config.ts` is all we needed, where we set our entry point.

```ts
/// <reference types="vite/client" />
import { resolve } from "node:path";
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";

export default defineConfig({
  plugins: [react()],
  build: {
    lib: {
      entry: resolve(__dirname, "src/index.ts"),
      formats: ["es"],
    },
  },
});
```

#### Dependencies

Another important piece of configuration here is to make sure we're not bundling up a bunch of dependencies that we don't need, such as React itself. We do this by extending `vite.config.ts` to include `build.rollupOptions` under both `external` and `output.globals`.

```ts
//...
export default defineConfig({
  plugins: //...,
  build: {
    lib: //...,
    rollupOptions: {
      external: ["react", "react-dom", "react/jsx-runtime"],
      output: {
        globals: {
          react: "React",
          "react-dom": "React-dom",
          "react/jsx-runtime": "react/jsx-runtime",
        },
      },
    },
  },
});
```

In my particular case, I was using `classnames` as a dependency for some styling, and ran into a few options when it came to ensuring any consumers of the library would have all the required dependencies.

**Bundled with the application**

Leaving `classnames` under `devDependencies` in `package.json` and simply having referenced it in the bundled code meant that `classnames` itself would also be bundled into the library itself. While there may be some cases where this makes sense, if the consuming application was using `classnames` already then we've essentially forced them to bundle the code in their application twice.

**Making it a regular dependency**

Moving `classnames` into the `dependencies` list in `package.json` might work, but that would lead to the consuming application needing to unnecessarily include `classnames` as a runtime dependency.

**Peer dependencies**

Having not seen this prior to the covered tutorials, it took some reasoning to understand what this was used for. Essentially, moving `classnames` into the `peerDependencies` would mean that it was a requirement of the library that the consuming application must have a matching version of the package installed. Thus, some leniency was required when specifying the version.

Installation was a concern that crossed my mind, however as of npm 7 [peer dependencies are installed by default](https://github.blog/news-insights/product-news/npm-7-is-now-generally-available/#peer-dependencies), meaning that a consumer need not go and manually install the peer dependencies themselves.

Ultimately I went with the **peer dependencies** approach as it seemed to have the fewest drawbacks.

#### Entry points and CSS

As mentioned in the tutorial, style sheets aren't automatically imported in the generated code and thus the consumer needs to import it themselves manually. This can be resolved by using the [vite-plugin-lib-inject-css](https://www.npmjs.com/package/vite-plugin-lib-inject-css) plugin. Once installed, it needs to be added to `vite.config.ts` under `plugins`.

This fixes our issue, however we now have a single import statement in our generated `index.js` file, meaning the entire style sheet needs to be imported if we use even a single component from our library. [Rollup recommends](https://rollupjs.org/configuration-options/#input) that we instead turn every file into an entry point, which will result in individual CSS files for each component &mdash; allowing us to import and use a single component, and only require that component's style sheet.

Adding the following to `build.rollupOptions` allows us to generate individual files for each component. The addition to `build.rollupOptions.output` is also necessary to retain our folder structure in the generated code.

```diff
export default defineConfig({
  build: {
    rollupOptions: {
+     input: Object.fromEntries(
+       glob
+         .sync('src/**/*.{ts,tsx}')
+         .map((file: string) => [
+           path.relative('src', file.slice(0, file.length - path.extname(file).length)),
+           fileURLToPath(new URL(file, import.meta.url))
+         ])
+     ),
      output: {
+       entryFileNames: '[name].js',
+       assetFileNames: 'assets/[name][extname]'
      },
    },
  },
})
```

This will result in the `dist/assets/` folder containing a CSS file for each component, which are imported accordingly.

#### Type generation

[vite-plugin-dts](https://www.npmjs.com/package/vite-plugin-dts) is the plugin required to generate our type declarations. Similar to `vite-plugin-lib-inject-css`, install it and add it to the `plugins` array in `vite.config.ts`.

Again bridging the two tutorials, I preferred the single-file approach taken by the first tutorial, and as such I added the plugin with the option `rollupTypes` set to `true`.

Assuming the `rollupTypes` option was enabled, the generated code should now contain an `index.d.ts` file with all of the types declared within it.

#### Setting up package.json

The other important file that requires some changes is the `package.json` file. Add or update the following fields.

```diff
{
+ "type": "module",
+ "files": ["dist"],
+ "module": "dist/index.js",
+ "types": "dist/index.d.ts"
}
```

- `type`: should be set to `module`. to indicate that we're using ES module syntax
- `files`: describes the files to be included when the package is published
- `module`: not an official Node feature, but supported by some bundlers
- `types`: exposes the type declarations entry point
- `exports`: _Optional_ the entry points to the library
- `main`: This is used to specify the entry point for `cjs`, which we're not supporting

**Life cycle scripts**

A script is also useful to specify here. While using `prepublishOnly` makes sense if we're planning to publish the library via `npm publish`, using `prepare` will run both on `npm publish` and `npm install` which allows us to install the library locally.

Add the following to `package.json` under `scripts`.

```diff
{
  "scripts": {
+   "prepare": "vite build"
  }
}
```

More information on these "life cycle scripts" can be found [here](https://arc.net/l/quote/qplfovup).

## Development

The following are simply some additions to the development tooling for the library itself. This won't be a guide on how to use any of the tooling, but simply provides some basic installation steps and ensuring that the files are not bundled into our generated library code.

### Storybook

Storybook can be installed by running the following command.

```bash
pnpm dlx storybook@latest init
```

The `src/stories/` folder can be removed if desired as it only contains some sample stories and documentation.

Stories can now be added for any of our components &mdash; documentation on how to do so can be found [here](https://storybook.js.org/docs/writing-stories). Once stories have been added, run Storybook with the following command.

```bash
pnpm storybook
```

**Ignoring Storybook files**

The last thing we need to do here is to ensure that we aren't bundling our stories in the generated code. We do this by extending the `glob.sync` command we added to `build.rollupOptions.input` in `vite.config.ts` and providing an `ignore` field to the options as follows.

```ts
glob.sync("src/**/*.{ts,tsx}", { ignore: ["src/**/*.stories.{ts,tsx}"] });
```

### Vitest

Vitest, jsdom and the React Testing Library (we'll need all three) can be installed with the following command.

```bash
pnpm i -D vitest jsdom @testing-library/react
```

Add a `test` script to `package.json`.

```diff
{
  "scripts": {
+   "test": "vitest"
  }
}
```

Next we need to update our `vite.config.ts` file.

```diff
+/// <reference types="vitest" />
export default defineConfig({
+ test: {
+   environment: "jsdom",
+   globals: true,
+   root: "src/",
+ },
});
```

And to get those globals working nicely so we don't need to repeatedly import `describe`, `test` etc. we need to add the following to `tsconfig.ts` under `compilerOptions`.

```diff
{
  "compilerOptions": {
+   "types": ["vitest/globals"]
  }
}
```

**Ignoring test files**

Similar to Storybook, we also need to make sure we're not generated code for our tests. Update the `glob.sync` command in `vite.config.ts` under `build.rollupOptions.input`.

```ts
glob.sync("src/**/*.{ts,tsx}", {
  ignore: ["src/**/*.stories.{ts,tsx}", "src/**/*.test.{ts,tsx}"],
});
```

## Using the library

Both of the linked tutorials go into publishing the library on npm, however due to the nature of a sample library I didn't want to delve into the publishing side of things.

However, outside of publishing it on npm I was unsure as to how I could actually use the library, and to my pleasure it was actually incredibly straightforward.

**Github repository**

Very simple, this allows you to install directly from the repository using the following command.

```bash
pnpm i -D GITHUB_NAME/REPOSITORY_NAME
```

Which results in the following `package.json` entry under `devDependencies`.

```json
{
  "PACKAGE_NAME": "github:GITHUB_NAME/REPOSITORY_NAME"
}
```

**Local reference**

Equally as straightforward, the following command can be run to create a local reference.

```bash
pnpm i -D PATH_TO_LIBRARY
```

Which similar to the above results in the following `package.json` entry under `devDependencies`.

```json
{
  "PACKAGE_NAME": "link:PATH_TO_LIBRARY"
}
```

### Sample repository

I did end up with a functional component library &mdash; albeit one with all of two components and a custom hook. It's available on GitHub [here](https://github.com/vivecuervo7/demolib), or it can be used as above with the following command.

```bash
pnpm i -D vivecuervo7/demolib
```
