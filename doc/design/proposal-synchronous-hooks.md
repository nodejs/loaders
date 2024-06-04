# Universal, synchronous and in-thread loader hooks

For background and motivation, see https://github.com/nodejs/node/issues/52219

TL;DR: the top priority of this proposal is to allow sun-setting CJS loader monkey patching as non-breakingly as possible (which is why it needs to be synchronous and in-thread because that's how the CJS loader and `require()` works). Then it's API consistency with the existing `module.register()` off-thread hooks.

Existing users of CJS loader monkey patching to look into:

- pirates (used by Babel and nyc)
- require-in-the-middle (used by many tracing agents)
- yarn pnp
- tsx
- ts-node

## API design

A high-level overview of the API.

```js
const { addHooks, removeHooks } = require('module');

function resolve(specifier, context, nextResolve) {
  const resolved = nextResolve(specifier, context);
  return { url: resolved.url.replaceAll(/foo/g, 'bar'); }
}

function load(url, context, nextLoad) {
  const loaded = nextLoad(specifier, context);
  return { source: loaded.source.toString().replaceAll(/foo/g, 'bar'); }
}

// id is a symbol
const id = addHooks({ resolve, load, ... });
removeHooks(id);
```

1. The names `addHooks` and `removeHooks` take inspiration from pirates. Can be changed to other better names.
2. An alternative design to remove the hooks could be `addHooks(...).unhook()`, in this case `addHooks()` returns an object that could have other methods.
   1. This may allow third-party hooks to query itself and avoid double-registering. Though this functionality is probably out of scope of the MVP.
3. It seems useful to allow the results returned by the hooks to be partial i.e. hooks don't have to clone the result returned by the next (default) hook to override it, instead they only need to return an object with properties that they wish to override. This can save the overhead of excessive clones.
4. It seems `shortCircuit` is not really necessary if hooks can just choose to not call the next hook?

## Hooks

### `resolve`: from specifier to url

```js
/**
 * @typedef {{
 *   parentURL?: string,
 *   conditions?: string[],
 *   importAttributes?: object[]
 * }} ModuleResolveContext
 */
/**
 * @typedef {{
 *   url?: string,
 *   format?: string
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
    return { url };
  }

  const resolved = nextResolve(specifier, context);
  if (resolved.url.endsWith('.zip')) {
    return { format: 'zip' };
  }
  return {};  // no override
}
```

Notes:

1. Example use case: yarn pnp ([esm hooks](https://github.com/yarnpkg/berry/blob/master/packages/yarnpkg-pnp/sources/esm-loader/hooks/resolve.ts), [cjs hooks](https://github.com/yarnpkg/berry/blob/master/packages/yarnpkg-pnp/sources/loader/applyPatch.ts))
2. `importAttributes` are only available when the module request is initiated with `import`.
3. For the CJS loader, `Module._cache` is keyed using file names, and it's used in the wild (usually in the form of `require.cache`) so changing it to key on URL would be breaking. We'll need to figure out another way to map it with an url.
   1. It might be non-breaking to maintain an additional map on the side for mapping the same url with different searches and hashes to a different module instance, or leave `Module._cache` as an alias map for the first module instance mapped by the URL minus search/hash. If in the same realm, `Module._cache` gets directly accessed from the outside, we can emit a warning about the incompatibility. Down the road this may require making `Module._cache` a proxy to propagate delete requests to the actual URL-based map.
   2. Changing CJS modules to be mapped with URL will have performance consequences from but might also help with the hot module replacement use case. Note that for parent packages with `exports` or `imports` in their `package.json` the URL conversion is already done (and not even cached) even in the CJS loader.

### `load`: from url to source code

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
function load(url, context, nextLoad) {
  const loaded = nextLoad(context);
  const { source: rawSource, format } = loaded;
  if (url.endsWith('.ts')) {
    const transpiled = ts.transpileModule(rawSource, {
      compilerOptions: { module: ts.ModuleKind.NodeNext }
    });

    return {
      format: 'commonjs',
      source: transpiled.outputText,
    };
  }

  return {};
}
```

Notes:

1. `context.format` is only present when the format is already determined by Node.js or a previous hook
2. It seems useful for the default load to always return a buffer, or add an option to `context` for the default hook to load it as a buffer, in case the resolved file point to a binary file (e.g. a zip file, a wasm, an addon). For the ESM loader it's (almost?) always a buffer. For CJS loader some changes are needed to keep the content in a buffer.
3. Some changes may be needed in both the CJS and ESM loader to allow loading arbitrary format in a raw buffer in a way that plays well with the internal cache and format detection.
4. This may allow us to finally deprecate `Module.wrap` properly

## `exports` (require-only): invoked after execution of the module

This only works for `require()` including `require(esm)`. It manipulates the exports object after execution of the original module completes. If the `exports` returned is not reference equal to the original exports object, it will affect later `module.exports` access in the original module but it does not affect direct `exports.foo` accesses (since the original exports are already passed through the context during module execution). If the module loaded is ESM (`context.format` is `module`) all the direct modification to `exports` are no-ops because ESM namespaces are not mutable. Returning a new `exports` for `require(esm)` is meaningless either - the only thing users can do is to read from the namespace, or to manipulate properties of the exported properties.

```js
/**
 * @typedef {{
 *  format: string
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
function exports(url, context, nextExports) {  // Mocking require-in-the-middle
  let { exports: originalExports } = nextExports(url, context);
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

This can only be meaningfully implemented for modules loaded by `require()`. ESM's design is completely different from CJS in that resolution of the graph and evaluation of the graph are separated, so a similar timing in ESM would be "after module evaluation". However, Node.js only gets to control the timing of the evaluation of the root module. The inner module evaluation is currently completely internal to V8. It may be possible to upstream a post-evaluation hook to V8, but calling from C++ to JS would also incur a non-trivial performance cost, especially if it needs to be done for every single module. Also, since the ESM namespace is specified to be immutable, what users can do after ESM evaluation is very limited - they cannot replace anything in the namespace or switch it to a different module. That's why the `link` hook was devised below to better work with the design of ESM (before linking completes it's possible to swap the module resolved by `import` to a different module).

### Alternative design: `requires` that encompasses `resolve` and `load`

An alternative design would be to span it across the `resolve` and `load` steps - take the specifier as argument, return exports in the result, and rename it to something like `requires()` (to avoid clashing with `require()`). The `nextRequires` hook would for example, invoke the default implementation of `requires` which in turn encompasses `resolve` and `load`. If `nextRequires` is not invoked then the default `resolve` and `load` will be skipped.

```js
function requires(specifier, context, nextRequires) {  // Mocking require-in-the-middle
  let { exports: originalExports, url } = nextRequires(specifier, context);
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

It's not yet investigated whether it is possible to make it work with the CJS loader cache at all, but it looks closer to what developers generally try to monkey-patch the CJS loader for.

## `link` (import-only): invoked before linking

This is invoked after `load` but prior to Node.js passing the final result to V8 for linking, so it can be composed with `resolve` and `load` if necessary. This needs to work with experimental vm modules for passing module instances around.

```js
/**
 * @typedef {{
 *  source: string | Buffer
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
function link(url, context, nextLink) {  // Mocking import-in-the-middle
  const { module: originalModule } = nextLink(url, context);
  assert.strictEqual(module.status, 'linked');  // Original module is linked at this point
  let source = `import * as original from 'original';`;
  source += `import { userCallback, name, basedir } from 'util'`;
  source += `const exported = {}`;
  for (const key of originalModule.namespace) {
    source += `let $${key} = original.${key};`;
    source += `export { $${key} as ${key} }`;
    source += `Object.defineProperty(exported, '${key}', { get() { return $${key}; }, set (value) { $${key} = value; }});`;
  }
  source += `userCallback(exported, name, basedir);`;
  const m = vm.SourceTextModule(source);
  m.linkSync((specifier) => {  // This is not yet implemented but should be trivial to implement.
    if (specifier === 'original') return originalModule;
    // Contains a synthetic module with userCallback, name & basedir computed from url
    if (specifier === 'util') return util;
  });
  return { module: m };
}
```

### Alternative design: `link` that encompasses `resolve` and `load`

An alternative design would be, instead of invoking a hook after `load`, make it wrap around `resolve` and `load` steps. The `link` hooks would take `specifier` and return `vm.Module` instances. If `nextLink()` is invoked, the next `resolve` and `load` (e.g. the default ones) will be invoked inside that. If `nextLink()` is not invoked, the default `resolve` and `load` will be skipped.

```js
function link(specifier, context, nextImport) {  // Mocking import-in-the-middle
  const { module: originalModule } = nextLink(specifier, context);
  assert.strictEqual(module.status, 'linked');  // Original module is linked at this point
  let source = '...';
  const m = vm.SourceTextModule(source);
  m.linkSync((specifier) => {
    // ...
  });
  return { module: m };
}
```

## Other use cases that may need some thought

1. Hot module replacement
2. Source map support (e.g. see [babel-register](https://github.com/babel/babel/blob/07bd0003cbdaa8525279c6dfa84e435471eb5797/packages/babel-register/src/hook.js#L38))
3. Virtual file system in packaged apps (single executable applications).
