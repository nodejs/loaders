# Moving Hooks on Thread

This is a proposal for how to move the existing hooks API back from a separate thread onto the same thread (main or worker) as the user’s application code, while preserving the ability for the hooks API to customize both CommonJS and ES modules.

## Motivation

The current off-thread design is proving very difficult to complete and maintain, and will not satisfy an important goal of sunsetting monkey-patching. The primary motivation for the off-thread design was so that users could write asynchronous customization hooks, but this seems to be a benefit that few users need; and the subset of users who need this ability can achieve it on their own, without such support needing to be built into the overall hooks API. Also, there are things that cannot be achieved without being on-thread, such as modifying exports from modules, that are important for instrumentation.

## Goals

1. Preserve the design of the current `resolve` and `load` hooks as much as possible, to ease migration for existing users of the off-thread hooks API.

1. Support most or all of the use cases currently covered by the off-thread hooks.

1. Complement @joyeecheung’s [Universal, synchronous and in-thread loader hooks proposal](https://github.com/nodejs/loaders/pull/198). We want one set of hooks for both CommonJS and ES modules, and that single set of hooks should both allow for both sunsetting monkey-patching and sunsetting the off-thread hooks.

## Design

This proposal builds on the [Universal, synchronous and in-thread loader hooks proposal](https://github.com/nodejs/loaders/pull/198). First we would implement the new `registerHooks` API from that proposal, which creates sync `resolve`, `load` and other hooks. That should cover that proposal’s primary goal of providing new APIs for monkey-patchers to migrate to, allowing us to sunset the need to support monkey-patching the CommonJS loader.

Once that is achieved, there is one large gap left to fill before we can remove the off-thread hooks: provide a way to perform async work, now that the hooks are all sync.

### `startGraph` hook (module graph entry point only)

We would create a new _async_ hook that runs for the main entry point, any worker entry point, and any dynamic `import()`: in other words, before any new module graph is created. This hook would delay any further hooks for that module graph operation, until the `startGraph` hook promise has resolved successfully. This provides an opportunity to do async work that the later sync hooks could have access to, for example to make network calls in a non-blocking way. One of the use cases for this hook is to support network imports, such as the [Import from HTTPS example](https://nodejs.org/api/module.html#import-from-https) in our docs. See [`nodejs/node` #43245](https://github.com/nodejs/node/pull/43245).

Because this hook might need to resolve and load the entire module graph to do its work, such as to resolve and fetch HTTPS URLs, it will be provided with the `resolve` and `load` hooks in its `context`. It would also receive `context.entry` to distinguish main/worker entry points from dynamic `import()`. Its signature would be something like:

```js
/**
 * @typedef {{
 *   parentURL?: string,
 *   entry: boolean,
 *   conditions?: string[],
 *   importAttributes?: Record<string, string>,
 *   resolve: Function,
 *   load: Function,
 * }} ModuleStartGraphContext
 */
/**
 * @typedef {Promise<{
 *   shortCircuit?: boolean
 * }>} ModuleStartGraphResult
 */
/**
 * @param {string} specifier
 * @param {ModuleStartGraphContext} context
 * @param {(specifier: string, context: ModuleStartGraphContext) => ModuleStartGraphResult} nextStartGraph
 * @returns {ModuleStartGraphResult}
 */
async function startGraph(specifier, context, nextStartGraph) {}
```

## Examples

### Import from HTTPS

the [Import from HTTPS example](https://nodejs.org/api/module.html#import-from-https) in our docs could use `startGraph` to handle the async work:

```js
import { parseAllImportSpecifiersFromSource } from 'some-library';

/** @type {Map<URL['href'], string | Buffer} */
const httpImportsCache = new Map();

export async function startGraph(specifier, context, nextStartGraph) {
  const { resolve, load } = context;
  const { url } = resolve(specifier, context);
  const urls = [url];
  while (urls.length > 0) {
    const url = urls.shift();
    if (url.startsWith('https://')) {
      const response = await fetch(url);
      const responseSource = await response.text();
      httpImportsCache.set(url, responseSource);
    }
    // load API constructs context automatically from registry data
    const { source } = load(url);
    httpImportsCache.set(url, source);
    for (const specifier of parseAllImportSpecifiersFromSource(source)) {
      // resolve API also constructs context automatically from registry data
      // including determining if the parent is CJS or ESM and setting conditions
      // correctly. Even when conditions is provided, although explicit import
      // or require conditions can override this process.
      const depUrl = resolve(specifier, { parentURL: url });
      urls.push(depUrl);
    }
  }
  return nextStartGraph(specifier, context);
}

export function resolve(specifier, context, defaultResolve) {
  if (specifier.startsWith('https://')) {
    return {
      url: specifier,
      shortCircuit: true,
    };
  }
  return defaultResolve(specifier, context);
}

export function load(url, context, defaultLoad) {
  const cached = httpImportsCache.get(url);
  if (cached) {
    return {
      source: cached,
      format: 'module', // This example assumes all fetched sources are modules for simplicity
      // No shortCircuit so that the load chain possibly transforms the source
    }
  }
  return defaultLoad(url, context);
}
```


### Transpilation

The [transpilation example from our docs](https://nodejs.org/api/module.html#transpilation) needs only to be changed so that async calls are now synchronous:

```js
// coffeescript-hooks.mjs
import { readFileSync } from 'node:fs';
import { dirname, extname, resolve as resolvePath } from 'node:path';
import { cwd } from 'node:process';
import { fileURLToPath, pathToFileURL } from 'node:url';
import coffeescript from 'coffeescript';

const extensionsRegex = /\.(coffee|litcoffee|coffee\.md)$/;

export function load(url, context, nextLoad) {
  if (extensionsRegex.test(url)) {
    const format = getPackageType(url);

    const { source: rawSource } = nextLoad(url, { ...context, format });
    const transformedSource = coffeescript.compile(rawSource.toString(), url);

    return {
      format,
      shortCircuit: true,
      source: transformedSource,
    };
  }

  return nextLoad(url);
}

function getPackageType(url) {
  const isFilePath = !!extname(url);
  const dir = isFilePath ? dirname(fileURLToPath(url)) : url;
  const packagePath = resolvePath(dir, 'package.json');
  try {
    const fileContents = readFileSync(packagePath, { encoding: 'utf8' })
    const { type } = JSON.parse(fileContents);
    if (type) { return type };
  } catch {}
  return dir.length > 1 && getPackageType(resolvePath(dir, '..'));
}
```

This could be used via `node --import 'data:text/javascript,import { registerHooks } from "node:module"; import { load } from "./coffeescript-hooks.mjs"; registerHooks({ load });' ./main.coffee`.
