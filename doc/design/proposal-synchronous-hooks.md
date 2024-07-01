# Universal, synchronous and in-thread loader hooks

For background and motivation, see https://github.com/nodejs/node/issues/52219. Prototype implementation is in https://github.com/joyeecheung/node/tree/sync-hooks.

TL;DR: the top priority of this proposal is to allow sun-setting CJS loader monkey patching as non-breakingly as possible (which is why it needs to be synchronous and in-thread because that's how the CJS loader and `require()` works). Then it's API consistency with the existing `module.register()` off-thread hooks.

Existing users of CJS loader monkey patching to look into:

- pirates (used by Babel and nyc)
- require-in-the-middle (used by many tracing agents)
- Yarn PnP
- tsx
- ts-node
- proxyquire, quibble, mockery

## API design

A high-level overview of the API.

```js
const { registerHooks } = require('module');

function resolve(specifier, context, nextResolve) {
  const resolved = nextResolve(specifier, context);
  resolved.url = resolved.url.replaceAll(/foo/g, 'bar');
  return resolved;
}

function load(url, context, nextLoad) {
  const loaded = nextLoad(specifier, context);
  loaded.source = loaded.source.toString().replaceAll(/foo/g, 'bar');
  return loaded;
}

const hook = registerHooks({ resolve, load });
hook.deregister(); // Calls hook[Symbol.dispose]()
```

* The name `registerHooks` and `deregister` can be changed.

## Hooks

### `resolve`: from specifier to url

```js
/**
 * @typedef {{
 *   parentURL?: string,
 *   conditions?: string[],
 *   importAttributes?: Record<string, string>,
 * }} ModuleResolveContext
 */
/**
 * @typedef {{
 *   url: string,,
 *   format?: string,
 *   shortCircuit?: boolean
 * }} ModuleResolveResult
 */
/**
 * @param {string} specifier
 * @param {ModuleResolveContext} context
 * @param {(specifier: string, context: ModuleResolveContext) => ModuleResolveResult} nextResolve
 * @returns {ModuleResolveResult}
 */
function resolve(specifier, context, nextResolve) {
  if (shouldOverride(specifier, context.parentURL)) {
    const url = customResolve(specifier, context.parentURL);
    return { url, shortCircuit: true };
  }

  const resolved = nextResolve(specifier, context);
  if (resolved.url.endsWith('.zip')) {
    resolved.format = 'zip';
    return resolved;
  }
  return resolved;  // no override
}
```

Notes:

1. Example use case: yarn pnp ([esm hooks](https://github.com/yarnpkg/berry/blob/master/packages/yarnpkg-pnp/sources/esm-loader/hooks/resolve.ts), [cjs hooks](https://github.com/yarnpkg/berry/blob/master/packages/yarnpkg-pnp/sources/loader/applyPatch.ts))
2. `importAttributes` are only available when the module request is initiated with `import`.
3. For the CJS loader, `Module._cache` is keyed using file names, and it's used in the wild (usually in the form of `require.cache`) so changing it to key on URL would be breaking. We'll need to figure out another way to map it with an url.
   1. It might be non-breaking to maintain an additional map on the side for mapping the same url with different searches and hashes to a different module instance, or leave `Module._cache` as an alias map for the first module instance mapped by the URL minus search/hash. If in the same realm, `Module._cache` gets directly accessed from the outside, we can emit a warning about the incompatibility. Down the road this may require making `Module._cache` a proxy to propagate delete requests to the actual URL-based map.
   2. Changing CJS modules to be mapped with URL will have performance consequences from but might also help with the hot module replacement use case. Note that for parent packages with `exports` or `imports` in their `package.json` the URL conversion is already done (and not even cached) even in the CJS loader.

### `load`: from url to source code

When `--experimental-network-import` is enabled, the default `load` step throws an error when encountering network imports. Although, users can implement blocking network imports by spawning a worker and use `Atomics.wait` to wait for the results. An example for doing blocking fetch synchronously can be found [here](https://github.com/addaleax/synchronous-worker?tab=readme-ov-file#how-can-i-avoid-using-this-package).

```js
/**
 * @typedef {{
 *   format?: string,
 *   conditions?: string[],
 *   importAttributes?: object[]
 * }} ModuleLoadContext
 */
/**
 * @typedef {{
*   format?: string,
*   source: string | Buffer
* }} ModuleLoadResult
*/
/**
 * @param {string} url
 * @param {ModuleLoadContext} context
 * @param {(context: ModuleLoadContext) => {ModuleLoadResult}} nextLoad
 * @returns {ModuleLoadResult}
 */

// Mini TypeScript transpiler:
function load(url, context, nextLoad) {
  const loaded = nextLoad(context);
  const { source: rawSource, format } = loaded;
  if (url.endsWith('.ts')) {
    const transpiled = ts.transpileModule(rawSource, {
      compilerOptions: { module: ts.ModuleKind.NodeNext }
    });

    loaded.format = 'commonjs';
    loaded.source = transpiled.outputText;
    return loaded;
  }

  return loaded;
}
```

Notes:

1. `context.format` is only present when the format is already determined by Node.js or a previous hook.
2. It seems useful for the default load to always return a buffer, or add an option to `context` for the default hook to load it as a buffer, in case the resolved file point to a binary file (e.g. a zip file, a wasm, an addon). For the ESM loader it's (almost?) always a buffer. For CJS loader some changes are needed to keep the content in a buffer.
3. Some changes may be needed in both the CJS and ESM loader to allow loading arbitrary format in a raw buffer in a way that plays well with the internal cache and format detection.
4. This may allow us to finally deprecate `Module.wrap` properly.
5. It may be useful to provide the computed extension in the context. An important use case is module format override (based on extensions?).

Example migration for Pirates:

```js
// Preserve Pirates’ existing public API where users call `addHook` to register a function
export function addHook(hook, options) {
  function load(url, context, nextLoad) {
    const loaded = nextLoad(url, context);
    const index = url.lastIndexOf('.');
    const ext = url.slice(index);
    if (!options.exts.includes(ext)) {
      return loaded;
    }
    const filename = fileURLToPath(url);
    if (!options.matcher(filename)) {
      return loaded;
    }
    loaded.source = hook(loaded.source, filename);
    return loaded;
  }

  const hook = module.registerHooks({ load });

  return function revert() {
    hook.deregister();
  };
}
```

## `exports` (require-only): from compiled CJS wrapper to exports object

This only runs for `require()` including `require(esm)`. It can only be meaningfully implemented for modules loaded by `require()`, since for ESM Node.js only gets to control the timing of the evaluation of the root module. The inner module evaluation is currently completely internal to V8. It may be possible to upstream a post-evaluation hook to V8, but calling from C++ to JS would also incur a non-trivial performance cost, especially if it needs to be done for every single module. Also, since the ESM namespace is specified to be immutable, what users can do after ESM evaluation is very limited - they cannot replace anything in the namespace or switch it to a different module. That's why the `link` hook was devised below to better work with the design of ESM (before linking completes it's possible to swap the module resolved by `import` to a different module).

Example mocking require-in-the-middle:

```js
/**
 * @typedef {{
 *  format: string,
 *  source: string | Buffer,
 * }} ModuleExportsContext
 */
/**
 * @typedef {{
*   exports: any
* }} ModuleExportsResult
*/
/**
 * @param {string} url
 * @param {ModuleExportsContext} context
 * @param {(context: ModuleExportsContext) => {ModuleExportsResult}} nextExports
 * @returns {ModuleExportsResult}
 */
function exports(url, context, nextExports) {
  let exported = nextExports(url, context);
  const stats = getStats(url);
  if (stats.name && modules.includes(stats.name)) {
    const newExports = userCallback(originalExports, stats.name, stats.basedir);
    return { exports: newExports }
  }
  return {
    exports: originalExports
  };
}
```

If the module loaded is CJS (`context.format` is `"commonjs"`), If the default step is run via `nextExports` run but the `exports` returned later is not reference equal to the original exports object, it will only affect later `module.exports` access in the original module, but does not affect contextual `exports.foo` accesses within the module. There are two options to address this:

1. We allow the default `exports` step to take an optional `context.exports`. If it's configured, it will be used during the wrapper invocation, which unifies the contextual access. Hooks will need to use a proxy if they want the new exports object to be based on the result from default evaluation (since by the time the new overridden exports object is created, the module hasn't been evaluated yet).
2. We expose the compiled CJS wrapper function in `context` so that users get to invoke it themselves and skip the default `exports` step.

1 may provide better encapsulation, because with 2 would mean that changes to the CJS wrapper function parameters becomes breaking.

If the module loaded is ESM (`context.format` is `"module"`) all the direct modification to `exports` are no-ops because ESM namespaces are not mutable. Returning a new `exports` for `require(esm)` only affects the caller of `require(esm)`, unless the hook returns a proxy to connect the changes between the new exports object and the module namespace object.

## `link` (import-only): from source code to compiled module records

This needs to work with experimental vm modules for passing module instances around.

Example mock of import-in-the-middle:

```js
/**
 * @typedef {{
 *  specifier: string,
 *  source: string | Buffer,
 * }} ModuleLinkContext
 */
/**
 * @typedef {{
*   module: vm.Module
* }} ModuleLinkResult
*/
/**
 * @param {string} url
 * @param {ModuleLinkContext} context
 * @param {(context: ModuleLinkContext) => {ModuleLinkResult}} nextLink
 * @returns {ModuleLinkResult}
 */
function link(url, context, nextLink) {
  const linked = nextLink(url, context);
  if (!shouldOverride(url, context.specifier)) {
    return linked;
  }
  assert.strictEqual(linked.module.status, 'linked');  // Original module is linked at this point
  let source = `import * as original from 'original';`;
  source += `import { userCallback, name, basedir } from 'util'`;
  source += `const exported = {}`;
  for (const key of linked.module.namespace) {
    source += `let $${key} = original.${key};`;
    source += `export { $${key} as ${key} }`;
    source += `Object.defineProperty(exported, '${key}', { get() { return $${key}; }, set (value) { $${key} = value; }});`;
  }
  source += `userCallback(exported, name, basedir);`;
  // This is not yet implemented but should be trivial to implement.
  linked.module = new vm.SourceTextModuleSync(source, function linkSync(specifier) {
    if (specifier === 'original') return linked;
    // Contains a synthetic module with userCallback, name & basedir computed from url
    if (specifier === 'util') return util;
  });
  return linked;
}
```

## Invocation order of `resolve` and `load` in ESM

With synchronous hooks, when overriding loading in a way may take a significant amount of time (for example, involving network access), the module requests are processed sequentially, losing the ability to make processing concurrent with a thread pool or event loop, which was what the design of ESM took care to make possible thanks to the `import` syntax. One trick can be used to start concurrent loading requests in `resolve` before blocking for the results in the `load` hook.

```js
function resolve(specifier, context, nextResolve) {
  const resolved = nextResolve(specifier, context);
  if (resolved.url.startsWith('http')) {
    // This adds a fetch request to a pool
    const syncResponse = startFetch(resolved.url);
    responseMap.set(url, syncResponse);
  }
  return resolved;
}

function load(url, context, nextLoad) {
  const syncResponse = responseMap.get(url);
  if (syncResponse) {
    const source = syncResponse.wait();
    return { source, shortCircuit: true };
  }
  return nextLoad(url, context);
}
```

However, to reuse `resolve` for concurrent loading, we need to implement it to be run in BFS order - that is, for a module like this:

```js
import 'a';
import 'b';
import 'c';
```

Instead of running the hooks as:

1. `resolve(a)` -> `load(a)` -> `link(a)`
2. `resolve(b)` -> `load(b)` -> `link(a)`
3. `resolve(c)` -> `load(c)` -> `link(a)`

We need to run the hook as:

1. `resolve(a)` -> `resolve(b)` -> `resolve(c)` (starts the off-thread fetching requests concurrently)
2. `load(a)` -> `load(b)` -> `load(c)` (block on the concurrently fetched results)
3. `link(a)` -> `link(b)` -> `link(c)`

Otherwise, we need to invent another hook that gets run in BFS order before both `resolve` and `load`, but in that case, the hook could only get the specifiers in its arguments, which may or may not be a problem. For network imports it's okay, because the specifiers are supposed to be URLs already, but other forms of concurrent processing (e.g. parallel transpilation with a worker pool) may not be possible without the result of `resolve`.

On the other hand, for CJS we have to run the hooks in DFS order since that's how CJS module requests have to be resolved.

## Child worker/process hook inheritance

Unlike the old `module.register()` design, this new API takes an object with functions so that it's possible to migrate existing monkey-patching packages over. One reason for the `path`/`url` parameter in the `module.register()` and the `initialize` was to aid hook inheritance in child worker and processes. With the new API, we can compose it differently by adding a new API specifically for registering a script that should be preloaded by child workers or processes. Hook authors can either put the call together with the hook registration (so that the inheritance is recursive automatically for grandchildren etc.), or put the hook registration code in a different file and configure a preload of that file.

```js
process.preloadInChildren(__filename);  // Or `import.meta.filename`
module.registerHooks({ load });
```

If the hook doesn't want it to be inherited in child workers or processes, it can opt out by not calling the preload API, or only call it under certain conditions.

## Additional use cases to consider

### Source map support

In most cases, source maps for Node.js apps are consumed externally (e.g. in a client to the inspector protocol) instead of in the Node.js instance that loads the modules, with the notable exception being error stack trace translation, enabled via `--enable-source-maps`. In this case, Node.js loads the source maps from disk using the URLs from the magic comments (currently, some of these are extracted using regular expressions, some are parsed by V8. It was discussed to rely on V8 instead of parsing them ourselves). To allow hooks to customize this behavior, we would need a hook that runs after module parsing and ideally before evaluation (otherwise, if an error is thrown during evaluation, the hook won't be able to run in time to affect the stack traces of the error being thrown). This can be done in the `link` or `exports` hooks proposed above if they take an optional property for source maps in the hook result, and the implementation can put the overriden sourcemap into the internal cache for later source position translations.

### Cache manipulation

It's common for users of existing CJS hooks to wipe the `require.cache` in their tests. Some tools may also update the `require.cache` for use cases like hot module replacement. This should probably be invented as a separate API to manipulate the cache (both the CJS and the ESM one).

