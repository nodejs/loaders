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
hook.deregister();  // Calls [Symbol.dispose](), or the other way around?
```

* The name `registerHooks` and `deregister` are subject to change.

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
3. For the CJS loader, `Module._cache` is keyed using filenames, and it's used in the wild (usually in the form of `require.cache`) so changing it to key on URL would be breaking. We'll attach a URL to the modules which can be converted to filenames or ids that are the same as their keys in `Module._cache` and use that in the hooks.

### `load`: from url to source code

Typically `load` is where hook authors might want to implement it asynchronously. In this case they can spawn their own worker and use `Atomics.wait()`  the hooks are synchronous, which has already been done by `module.register()`'s off-thread hooks anyway, and this new API is just giving the control/choice back to the hook authors. An example for doing blocking fetch synchronously using workers can be found [here](https://github.com/addaleax/synchronous-worker?tab=readme-ov-file#how-can-i-avoid-using-this-package).

When the hooks only access the file system and do not access the network, it is recommended to just use the synchronous FS APIs, which has lower overhead compared to the asynchronous versions.

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
 *   format: string,
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
2. It seems useful for the default load to always return a buffer, or add an option to `context` for the default hook to load it as a buffer, in case the resolved file point to a binary file (e.g. a zip file, a wasm, an addon). For the ESM loader it's (almost?) always a buffer. For CJS loader some changes are needed to keep the content in a buffer. For now users need to assume that the source could be either string or a buffer and need to handle both, this is also the case for existing `module.register()` hooks.
3. This may allow us to finally deprecate `Module.wrap` properly.
4. It may be useful to provide the computed extension in the context. An important use case is module format override (based on extensions?)

Example migration for the pirates package:

```js
// Preserve pirates’ existing public API where users call `addHook` to register a function
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
 *  module: module.Module
 * }} ModuleExportsContext
 */
/**
 * @typedef {{
*   exports: any,
*   shortCircuit?: boolean
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

## `instantiate` (import-only): from compiled ES modules to instantiated ES modules (with dependency resolved)

Previously with only the `load` hooks for overriding module behavior, hooks typically have to parse the provided source again to understand the exports, which can be both slow and prone to bugs. `instantiate` gives the hooks an opportunity to override module behavior using the export names parsed by V8 and perform customization before the the compiled code gets evaluated. For ESM, it needs to work with experimental vm modules for passing customized module instances around.

TODO(joyeecheung): we may consider exposing `compile` (source code to compiled modules) too, which happens before `instantiate` and can be overloaded for CJS as well. The `instantiate` hook has to be separate because merely compiling the modules is not enough for V8 to provide the export names. V8 requires the `import` dependencies to be resolved before it can return the namespace, since there might be `export * from '...'` coming from the dependencies.

Example mock of import-in-the-middle:

```js
/**
 * @typedef {{
 *  specifier: string,
 *  module: vm.Module,
 * }} ModuleInstantiateContext
 * ModuleWrap can be too internal to be exposed. We'll need to wrap it with a more public
 * type before passing it to users. However, users should not expect the wrapper to own
 * the ModuleWrap/V8 module records or use this for memory management.
 */
/**
 * @typedef {{
*   module?: vm.Module,
*   shortCircuit?: boolean
* }} ModuleInstantiateResult
*/
/**
 * @param {string} url
 * @param {ModuleInstantiateContext} context
 * @param {(context: ModuleInstantiateContext) => {ModuleInstantiateResult}} nextInstantiate
 * @returns {ModuleInstantiateResult}
 */
function instantiate(url, context, nextInstantiate) {
  const instantiated = nextInstantiate(url, context);
  if (!instantiated.module || !shouldOverride(url, context.specifier)) {  // Only overrides ESM.
    return instantiated;
  }
  assert.strictEqual(instantiated.module.status, 'instantiated');  // Original module has resolved its dependencies at this point
  let source = `import * as original from 'original';`;
  source += `import { userCallback, name, basedir } from 'util'`;
  source += `const exported = {}`;
  for (const key of instantiated.module.namespace) {  // Note how instantiated.module.namespace is the real exports parsed by V8.
    source += `let $${key} = original.${key};`;
    source += `export { $${key} as ${key} }`;
    source += `Object.defineProperty(exported, '${key}', { get() { return $${key}; }, set (value) { $${key} = value; }});`;
  }
  source += `userCallback(exported, name, basedir);`;
  // This is not yet implemented but should be trivial to implement.
  instantiated.module = new vm.SourceTextModuleSync(source, function linkSync(specifier) {
    if (specifier === 'original') return instantiated;
    // Contains a synthetic module with userCallback, name & basedir computed from url
    if (specifier === 'util') return util;
  });
  assert.strictEqual(instantiated.module.status, 'instantiated');
  return instantiated;
}
```

Alternatively, for overriding the compiled module instance with a `SourceTextModule` that doesn't have additional dependencies itself, we can assume that if `result.module` is a string for a `SourceTextModule` that needs to be recompiled using the original configurations, and perform the recompilation without requiring users to use `vm.SourceTextModule`.

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

## The API takes functions directly, delegating worker inheritance to another API

Unlike the old `module.register()` design, this new API takes an object with functions so that it's possible to migrate existing monkey-patching packages over, for several reasons:

1. It's unclear how an API that takes a `specifier` & `parentURL` combination should resolve the request in a universal manner - `module.register()` simply reuses the `import` resolution which is assuming too much for universal hooks e.g. `import` conditions in `package.json` would be used over other conditions, which can be surprising if the hooks are supposed to be `require()`-ed. ESM resolution is deliberately different from CJS resolution, so choosing either of them or trying to invent a new resolution rule adds unnecessary quirks to the API, it's better to just leave the choice to users.
2. The purpose of the `specifier` & `parentURL` combination in `module.register()` was to help child worker inherit the hooks. In practice, however, inheriting the module request pointed by the registration API alone is not necessarily enough for the entire customization to work. For example, in a typical `instrumentation.js` -> `@opentelemetry/instrumentation` -> `import-in-the-middle` -> `import-in-the-middle/hook.mjs` customization dependency chain, only automatically preloading `import-in-the-middle/hook.mjs` doesn't complete customization for workers if the higher-level code don't get preloaded too. In the end the high-level code would still need a way to register themselves as preloads for child workers, making the inheritance of the low-level hooks alone redundant.

So this new API will simply take the functions directly and delegate the worker inheritance/preload registration to a separate API. Currently, existing hooks users have been using `--require`/`--import` and the `process.execArgv` inheritance in workers to deal with registration of the entire customization, and this will continue to work with the new API. In the future we can explore a few other options:

1. A file-based configuration to specify preloads that are automatically discovered by Node.js upon startup. See [the `noderc` proposal](https://github.com/nodejs/node/issues/53787).
2. A JavaScript API to register preloads in child workers. For example:

    ```js
    // As if `require(require.resolve(specifier, { paths: require.resolve.paths(base) }))`
    // is used to preload the file.
    process.requireInChildren(specifier, base);
    // As if import(new URL(specifier, parentURL)) is used to preload the file.
    process.importInChildren(specifier, parentURL);
    ```

 This API is supposed to be invoked by final customizations or end user code, or intermediate dependency that wish to spawn a worker with different preloads, because only they have access to the entire graph of the customizations that should be inherited into the workers. For example, in the case of a typical open-telemetry user, the file being preloaded should be their `instrumentation.js`. We could also add accompanying APIs for removing/querying preloads.

## Additional use cases to consider

### Source map support

In most cases, source maps for Node.js apps are consumed externally (e.g. in a client to the inspector protocol) instead of in the Node.js instance that loads the modules, with the notable exception being error stack trace translation, enabled via `--enable-source-maps`. In this case, Node.js loads the source maps from disk using the URLs from the magic comments (currently, some of these are extracted using regular expressions, some are parsed by V8. It was discussed to rely on V8 instead of parsing them ourselves). To allow hooks to customize this behavior, we would need a hook that runs after module parsing and ideally before evaluation (otherwise, if an error is thrown during evaluation, the hook won't be able to run in time to affect the stack traces of the error being thrown). This can be done in the `link` or `exports` hooks proposed above if they take an optional property for source maps in the hook result, and the implementation can put the overriden sourcemap into the internal cache for later source position translations.

### Cache manipulation

It's common for users of existing CJS hooks to wipe the `require.cache` in their tests. Some tools may also update the `require.cache` for use cases like hot module replacement. This should probably be invented as a separate API to manipulate the cache (both the CJS and the ESM one).

### Locking the module hooks

To prevent accidental mistakes or to aid analysis of the dependency, it may be helpful to provide an API that stops further module hook registration after its invocation.
