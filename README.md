# Loaders Team

## Purpose

The Node.js Loaders Team maintains and actively develops the ECMAScript Modules Loaders implementation in Node.js core.

## History

This team is spun off from the [Modules team](https://github.com/nodejs/modules). We aim to implement the [use cases](https://github.com/nodejs/modules/blob/main/doc/use-cases.md) that went unfulfilled by the initial ES modules implementation that can be achieved via module customization hooks; to provide hooks that are sufficiently powerful to allow the sunset of CommonJS monkey-patching; and to improve the ES modules implementation to reach parity with CommonJS.

## Project

- [Resources](doc/resources.md)

- [Use cases](./doc/use-cases.md)

- [Design](./doc/design/overview.md)

- [Project board](https://github.com/nodejs/node/projects/17)

- [Meeting minutes](./doc/meetings)

## Status

### Goal: Sunset Monkey-Patching

- [ ] Synchronous version of the module customization hooks so that the hooks can be used in the `require` path on the main thread, per [design](./doc/design/proposal-synchronous-hooks.md).

### Goal: Parity with CommonJS

- [ ] Synchronize the ESM loader, to hopefully close the startup performance gap with CommonJS and allow more predictable async activity on startup (for example, no `async_hooks` activity will precede user code). [Tracking issue](https://github.com/nodejs/node/issues/55782).

- [ ] Implement `import.meta.main`, matching other runtimes. [Tracking issue](https://github.com/nodejs/node/issues/57226).

### Other Improvements

- [ ] First-class support for [import maps](https://github.com/WICG/import-maps) that doesn’t require a custom loader.

- [ ] Add helper/utility functions to reduce boilerplate in user-defined hooks.

   - [ ] Start with helpers for retrieving the closest parent `package.json` associated with a specifier string; and for retrieving the `package.json` for a particular package by name (which is not necessarily the same result).

   - [ ] Potentially include all the functions that make up the ESM resolution algorithm as defined in the [spec](https://nodejs.org/api/esm.html#resolution-algorithm-specification). Create helper functions for each of the functions defined in that psuedocode: `esmResolve`, `packageImportsResolve`, `packageResolve`, `esmFileFormat`, `packageSelfResolve`, `readPackageJson`, `packageExportsResolve`, `lookupPackageScope`, `packageTargetResolve`, `packageImportsExportsResolve`, `patternKeyCompare`. (Not necessarily all with these exact names, but corresponding to these functions from the spec.)

   - [ ] Follow up with similar helper functions that make up what happens within Node’s internal `load`. (Definitions to come.)

- [ ] Helper/utility functions to allow access to the CommonJS named exports discovery algorithm (`cjs-module-lexer`).

- [ ] Hooks for customizing the REPL, including transpilation and tab completion. Support users pasting TypeScript (or CoffeeScript or whatever) into the REPL and having just as good an experience as with plain JavaScript.

   - [ ] Support top-level `await` in the REPL API, if possible.

- [ ] Allow customizing string inputs: `--eval` CLI flag, `Worker` constructor, stdin, etc.

- [ ] Hooks for customizing the stack trace (in other words, a hook version of `Error.prepareStackTrace`). This would allow transpiled languages to improve the output.

- [ ] Hooks for customizing filesystem calls, for allowing things like virtual filesystems or archives treated as volumes.

- [ ] Inherit configuration blob to worker threads and child processes.
