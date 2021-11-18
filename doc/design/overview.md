# Loaders Design

There are currently the following [loader hooks](https://github.com/nodejs/node/blob/HEAD/doc/api/esm.md#hooks):

## Basic hooks

1. `resolve`: Takes a specifier (the string after `from` in an `import` statement) and converts it into an URL to be loaded.

1. `load`: Takes the resolved URL and returns runnable code (JavaScript, Wasm, etc.) as well as the name of one of Node’s ESM loader’s [“translators”](https://github.com/nodejs/node/blob/master/lib/internal/modules/esm/translators.js):
   * `commonjs`
   * `module`
   * `builtin` (a Node internal module, like `fs`)
   * `json` (with `--experimental-json-modules`)
   * `wasm` (with `--experimental-wasm-modules`)

## Filesystem hooks

The Node resolution algorithms may rely on various filesystem operations in order to return definite answers. For example, in order to know whether the package `foo` resolves to `/path/to/foo/index.js`, one must first check the [`exports` field](https://nodejs.org/api/packages.html#exports) located in `/path/to/foo/package.json`. Similarly, a loader that would add support for import maps need to know how to retrieve those import maps in the first place.

While this is fairly easy when operating with the traditional filesystem (one could just use the `fs` module), things get trickier when you consider that loaders may also have to deal with other data sources. For instance, a loader that would import files directly from the network (similar to how Deno operates) would be unable to leverage `fs` to access the `package.json` content for the remote packages. Same thing when the package data are kept within archives that would require special support for access (like Electron or Yarn both operate).

To facilitate such interactions between loaders, they are given the ability to override the basic filesystem operations used by the Node resolution helpers. This way, they can remain blissfully unaware of the underlying data source (filesystem or network or otherwise) and focus on the part of the resolution they care about.

1. `statFile`: Takes the resolved URL and returns its [`fs.Stats` record](https://nodejs.org/api/fs.html#class-fsstats) (or `null` if it doesn't exist).

1. `readFile`: Takes the resolved URL and returns its binary content (or `null` if it doesn't exist).

## Advanced hooks

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
