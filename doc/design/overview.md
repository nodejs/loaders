# Loaders Design

There are currently [three loader hooks](https://github.com/nodejs/node/tree/master/doc/api/esm.html#esm_hooks):

1. `resolve`: Takes a specifier (the string after `from` in an `import` statement) and converts it into an URL to be loaded.

1. `load`: Takes the resolved URL and returns runnable code (JavaScript, Wasm, etc.) as well as the name of one of Node’s ESM loader’s [“translators”](https://github.com/nodejs/node/blob/master/lib/internal/modules/esm/translators.js):
   * `commonjs`
   * `module`
   * `builtin` (a Node internal module, like `fs`)
   * `json` (with `--experimental-json-modules`)
   * `wasm` (with `--experimental-wasm-modules`)

* `globalPreload`: Defines a string of JavaScript to be injected into the application global scope.

## History

### Hook consolidation

The initial experimental implementation consisted of 5 hooks:

* `resolve`
* `getFormat`
* `getSource`
* `transformSource`
* `getGlobalPreloadCode`

These were consolidated (in nodejs/node#37468) to avoid counter-intuitive and paradoxical behaviour when used in multiple custom loaders.

## Chaining

Custom loaders are intended to chain to support various concerns beyond the scope of core, such as build tooling, mocking, transpilation, etc.

### `globalPreload`

For now, we think that this wouldn’t be chained the way `resolve` and `load` would be. This hook would just be called sequentially for each registered loader, in the same order as the loaders themselves are registered. If this is insufficient, for example for instrumentation use cases, we can discuss and potentially change this to follow the chaining style of `load`.

### `resolve`

### `load`

Chaining `load` hooks would be similar to chaining `resolve` hooks, where `source` is the loaded module’s source code/contents and `format` is the name of one of Node’s ESM loader’s [“translators”](https://github.com/nodejs/node/blob/master/lib/internal/modules/esm/translators.js): `builtin` (a Node internal module like `fs`), `commonjs`, `json` (with `--experimental-json-modules`), `module`, or `wasm` (with `--experimental-wasm-modules`).

Currently, Node’s internal ESM loader throws an error on unknown file types: `import('file.javascript')` throws, even if the contents of `file.javascript` are perfectly acceptable JavaScript. This error happens during Node’s internal `resolve` when it encounters a file extension it doesn’t recognize; hence the current [CoffeeScript loader example](https://nodejs.org/api/esm.html#esm_transpiler_loader) has lots of code to tell Node to allow CoffeeScript file extensions. We should move this validation check to be after the format is determined, which is one of the return values of `load`; so basically, it’s the responsibility of `load` to return a `format` that Node recognizes. Node’s internal `load` doesn’t know to resolve a URL ending in `.coffee` to `module`, so Node would continue to error like it does now; but the CoffeeScript loader under this new design no longer needs to hook into `resolve` at all, since it can determine the format of CoffeeScript files within `load`.

### Proposals

* [Recursive chaining](./proposal-chaining-recursive.md)
* [Iterative chaining](./proposal-chaining-iterative.md)
