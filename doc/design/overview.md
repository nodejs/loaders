# Loaders Design

There are currently the following [loader hooks](https://github.com/nodejs/node/blob/master/doc/api/esm.md#hooks):

1. `resolve`: Takes a specifier (the string after `from` in an `import` statement) and converts it into an URL to be loaded.

1. `load`: Takes the resolved URL and returns runnable code (JavaScript, Wasm, etc.) as well as the name of one of Node’s ESM loader’s [“translators”](https://github.com/nodejs/node/blob/master/lib/internal/modules/esm/translators.js):
   * `commonjs`
   * `module`
   * `builtin` (a Node internal module, like `fs`)
   * `json` (with `--experimental-json-modules`)
   * `wasm` (with `--experimental-wasm-modules`)

1. `statFile`: Takes the resolved URL and returns its [`fs.Stats` record](https://nodejs.org/api/fs.html#class-fsstats) (or `null` if it doesn't exist).

1. `readFile`: Takes the resolved URL and returns its binary content (or `null` if it doesn't exist).

1. `globalPreload`: Defines a string of JavaScript to be injected into the application global scope.

## Chaining

Custom loaders are intended to chain to support various concerns beyond the scope of core, such as build tooling, mocking, transpilation, etc.

### Proposals

* [Chaining Hooks “Iterative” Design](./proposal-chaining-iterative.md)
* [Chaining Hooks “Middleware” Design](./proposal-chaining-middleware.md)

## History

### Hook consolidation

The initial experimental implementation consisted of 5 hooks:

* `resolve`
* `getFormat`
* `getSource`
* `transformSource`
* `getGlobalPreloadCode`

These were consolidated (in [nodejs/node#37468](https://github.com/nodejs/node/pull/37468)) to avoid counter-intuitive and paradoxical behaviour when used in multiple custom loaders.
