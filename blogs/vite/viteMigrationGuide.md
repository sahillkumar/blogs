---
title: Migrating from Create React App to Vite — A Step-by-Step Guide
published: true # if you set this to false it will publish the page as a draft
description: Create React App to Vite Migration
tags: "react, CRA, webpack, vite"
canonical_url: null
id: 2369138
---

# Migrating from Create React App to Vite — A Step-by-Step Guide

Migrating from Create React App (CRA) to Vite enhances development speed and efficiency, making it ideal for industry-level projects. CRA’s reliance on Webpack often results in slower builds and hot module replacement (HMR). Vite, leveraging ES modules, provides instant HMR, faster builds, and a leaner setup. This guide walks through the migration process, covering essential updates for configurations, environment variables, TypeScript, testing, and build optimizations.

### 1. Install Required Packages

First, install the necessary dependencies for Vite:

```sh
npm install vite @vitejs/plugin-react --save-dev
```

- **`vite`** is the core build tool and development server that provides fast bundling and HMR.
- **`@vitejs/plugin-react`** enables React-specific optimizations like Fast Refresh and JSX transformation.

### 2. Create Vite Configuration File

Create a **`vite.config.ts`** file in the root directory:

```ts
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";

export default defineConfig({
  plugins: [react()],
  server: {
    port: 3000,
    open: true,
  },
});
```

- **`server.port`** specifies the port on which the Vite development server will run. By default, Vite uses port **`5173`**, by setting it to **`3000`** makes it consistent with CRA.
- **`server.open`** automatically opens the browser when the development server starts.

### 3. Move `index.html` to Root

Move **`public/index.html`** to the project root and update **`src`**:

```html
<script type="module" src="./src/index.jsx"></script>
```

Unlike Webpack, which dynamically injects the **`src`** attribute into **`index.html`** using plugins like **`HtmlWebpackPlugin`**, Vite requires you to explicitly define the entry file. Since Vite does not modify **`index.html`** dynamically, specifying the **`src`** ensures the application loads correctly.

Now, remove all **`%PUBLIC_URL%`** occurrences in the index.html file:

```html
<!-- CRA -->
<link rel="icon" href="%PUBLIC_URL%/favicon.ico" />

<!-- VITE -->
<link rel="icon" href="/favicon.ico" />
```

### 4. Update Script Commands

Replace the CRA scripts in **`package.json`** with Vite equivalents:

```json
"scripts": {
  "start": "vite",
  "build": "vite build",
}
```

### 5. Update Environment Variables

Vite exposes env variables under **`import.meta.env`** object as strings automatically. To prevent accidentally leaking env variables to the client, only variables prefixed with **`VITE_`** are exposed to your Vite-processed code.

##### Update Variable Declarations:

Replace **`REACT_APP_`** with **`VITE_`** in all your environment variables

```ts
// CRA
REACT_APP_STAGING = staging;

// VITE
VITE_STAGING = staging;
```

##### Update Variable Usage:

```ts
// CRA
export const config = process.env.REACT_APP_STAGING ? "staging" : "dev";

// VITE
export const config = import.meta.env.VITE_STAGING ? "staging" : "dev";
```

### 6. Improve TypeScript Support and `tsconfig.json`

If your project uses TypeScript, follow these steps to ensure compatibility and improved performance.

Update the following properties in `tsconfig.json`:

```json
{
  "compilerOptions": {
    "moduleResolution": "bundler",
    "isolatedModules": true,
    "types": ["vite/client"]
  },
  "include": ["vite.config.ts"]
}
```

One thing you should know about Vite is that Vite only performs transpilation on **`.ts`** files and does **NOT** perform type checking. It assumes type checking is taken care of by your IDE and build process.

The reason Vite does not perform type checking as part of the transform process is because the two jobs work fundamentally differently. Transpilation can work on a per-file basis and aligns perfectly with Vite's on-demand compile model. In comparison, type checking requires knowledge of the entire module graph. Shoe-horning type checking into Vite's transform pipeline will inevitably compromise Vite's speed benefits.

##### Adding type checking support in Development

During development, if you need more than IDE hints, Vite recommends running **`tsc --noEmit --watch`** in a separate process, or use **`vite-plugin-checker`** if you prefer having type errors directly reported in the browser. In this guide we will go with the second approach.

Install **`vite-plugin-checker`**:

```sh
npm install --save-dev vite-plugin-checker
```

Update **`vite.config.ts`**:

```ts
import tsChecker from "vite-plugin-checker";

export default defineConfig({
  plugins: [tsChecker({ typescript: true })],
});
```

##### Adding type checking support for Production Builds

For production builds, we will run **`tsc --noEmit`** in addition to Vite's build command.
Update **`package.json`**:

```json
"scripts": {
  "build": "tsc --noEmit && vite build",
}
```

> The **`--noEmit`** flag prevents the compiler from generating JavaScript output files.

### 7. Convert Files to `.jsx` and `.tsx`

Rename files accordingly:

- **`.js`** → **`.jsx`**
- **`.ts`** → **`.tsx`**

<details>
  <summary>Why is this required ?</summary>
    Vite avoids processing all <strong><code>.js</code></strong> and <strong><code>.ts</code></strong> files for JSX or TypeScript syntax by default to prevent unnecessary overhead, which could slow down the development server and build times. Instead, by explicitly using <strong><code>.jsx</code></strong> and <strong><code>.tsx</code></strong> extensions, Vite selectively applies transformations only to the files that require them. When encountering these files, Vite leverages esbuild to efficiently transform JSX or TypeScript syntax into standard JavaScript, ensuring faster builds and a more optimized development workflow.
</details>

### 8. Add Absolute Path Support & Path Aliasing

In the Vite configuration file, paths can be resolved using the config.resolve property. Since manually defining numerous paths can be tedious, we use the vite-tsconfig-paths package. This package enables Vite to resolve imports using TypeScript’s path mapping automatically.

Install **`vite-tsconfig-paths`**:

```sh
npm install --save-dev vite-tsconfig-paths
```

Update **`vite.config.ts`**:

```ts
import tsconfigPaths from "vite-tsconfig-paths";
import path from "path";

export default defineConfig({
  plugins: [tsconfigPaths()],
  resolve: {
    alias: {
      "@assets": path.resolve(__dirname, "src/assets"),
    },
  },
});
```

If any of your path aliases do not work as expected with **`vite-tsconfig-paths`**, you can manually define them as shown above.

### 9. Add SVG Support

Install **`vite-plugin-svgr`**:

```sh
npm install --save-dev vite-plugin-svgr
```

Update **`vite.config.ts`**:

```ts
import svgr from "vite-plugin-svgr";

export default defineConfig({
  plugins: [svgr()],
});
```

##### Import SVG files as components:

```js
import ChartIcon from "@assets/icons/Charts.svg?react";
```

### 10. Configure Production Build & Target

Update **`vite.config.ts`**:

```ts
export default defineConfig({
  build: {
    outDir: "build",
    target: "es2021",
  },
});
```

- **`build.outDir`** specifies the output directory for the built files. By default, Vite outputs the build to the dist folder, but here it is customized to match CRA's default output directory.

- **`build.target`** sets the JavaScript target version for the output files. The default value is "modules", ensuring compatibility with modern browsers that support ES module syntax.
- For older browser compatibility, refer to the [Vite Browser Compatibility Guide](https://vite.dev/guide/build.html#browser-compatibility)

### 11. Add Testing with Vitest

Install testing dependencies:

```sh
npm install --save-dev vitest jsdom @testing-library/react @testing-library/jest-dom @vitest/ui
```

Update **`tsconfig.json`**:

```json
"compilerOptions": {
    "types": ["vitest/globals"]
},
```

Update **`package.json`**:

```json
"scripts": {
  "test": "vitest --ui"
}
```

- The **`--ui`** flag in **`vitest --ui`** starts the Vitest UI, providing a visual test runner interface. It allows you to view test results, rerun specific tests, and debug failing tests interactively within a browser-based interface.

Update **`vite.config.ts`**:

```ts
export default defineConfig({
  test: {
    globals: true,
    environment: "jsdom",
    setupFiles: /* <path to test setup file> */,
    include: ["src/**/*.test.*"],
  },
});
```

Update test setup file:

```ts
import { expect } from "vitest";
import matchers from "@testing-library/jest-dom/matchers";

// extends Vitest's expect method with methods from react-testing-library
expect.extend(matchers);
```

Update Test Files:

- **`describe`**, **`expect`**, **`it`**, and **`test`** work the same as in Jest
- Replace **`jest.fn()`** with **`vi.fn()`**
- For further changes, refer to [Vitest Migration Guide](https://vitest.dev/guide/migration.html#jest)

### 12. Adding Code Coverage

We will integrate **Istanbul** to generate code coverage reports.

Installation:

```sh
npm install --save-dev @vitest/coverage-istanbul
```

Update Script in **`package.json`**:

```json
"scripts": {
  "test:coverage": "vitest run --coverage"
}
```

Update **`vite.config.ts`**:

```ts
export default defineConfig({
  test: {
      coverage: {
        provider: "istanbul",
        reporter: ["lcov"],
        include: /* Specify directory or files here */,
      },
    },
});
```

### 13. Enable Source Maps & Add Bundle Visualizer

To enable source maps in Vite, update the **`build.sourcemap`** option in the Vite configuration:

Update **`vite.config.ts`**:

```ts
export default defineConfig({
  build: {
    sourcemap: true,
  },
});
```

Since Vite uses **Rollup** for production builds, we will install **`rollup-plugin-visualizer`** to visualize source maps.

Install **`rollup-plugin-visualizer`**:

```
npm install --save-dev rollup-plugin-visualizer
```

Update **`vite.config.ts`**:

```ts
import { visualizer } from "rollup-plugin-visualizer";

export default defineConfig({
  plugins: [visualizer()],
});
```

To prevent the visualizer from running with every build, we will create a separate script for it.
Update **`package.json`**:

```sh
"scripts": {
  "stats": "vite build && open stats.html"
}
```

### 14. Setting Global Variable

Vite does not define a global field on **`window`** like Webpack does. Some libraries rely on this behavior because Webpack, being older, has established it as a common practice.

Update **`index.html`**:

```html
<script>
  window.global = globalThis;
</script>
```

### 15. Remove CRA and Webpack-Related Configurations

Finally, delete unnecessary Webpack-related files and dependencies, including:

- Remove **`react-scripts`** commands
- Uninstall Jest-related packages
- Uninstall the following Webpack-related dependencies:
  - **`webpack`**
  - **`webpack-bundle-analyzer`**

### References

- [Vite](https://vitejs.dev/)
- [Vitest](https://vitest.dev/)
- [Migrate to Vite from Create React App (CRA)](https://www.robinwieruch.de/vite-create-react-app/)

### Contribute

If you feel any step can be improved, more details can be added, or new steps should be included, please feel free to contribute!

Check out the repository and submit a pull request:  
[GitHub Repository](https://github.com/sahillkumar/blogs/blob/main/blogs/vite/viteMigrationGuide.md)
