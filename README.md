# Loaders Team

## Purpose

The Node.js Loaders Team maintains and actively develops the ECMAScript Modules Loaders implementation in Node.js core.

## History

This team is spun off from the [Modules team](https://github.com/nodejs/modules). We aim to implement the [use cases](https://github.com/nodejs/modules/blob/main/doc/use-cases.md) that went unfulfilled by the initial ES modules implementation that can be achieved via loaders.

## Project

- [Resources](doc/resources.md)

- [Use cases](./doc/use-cases.md)

- [Design](./doc/design/overview.md)

- [Project board](https://github.com/nodejs/node/projects/17)

- [Meeting minutes](./doc/meetings)

## Status

### Milestone 1: Parity with CommonJS

Before extending into new frontiers, we need to improve the loaders API enough that users can do just about everything they could do in CommonJS with ESM + loaders. (Outside of loaders scope, but related to the goal of parity between CommonJS and ESM, is finishing and stabilizing `--experimental-vm-modules`.)

- [x] Finish https://github.com/nodejs/node/pull/37468 / https://github.com/nodejs/node/pull/35524, simplifying the hooks to `resolve`, `load` and `globalPreloadCode`.

- [x] Refactor the internal Node ESMLoader hooks into `resolve` and `load`. Node’s internal loader already has no-ops for `transformSource` and `getGlobalPreloadCode`, so all this really entails is wrapping the internal `getFormat` and `getSource` with one function `load` (`getFormat` is used internally outside ESMLoader, so they cannot merely be merged). https://github.com/nodejs/node/pull/37468

- [x] Refactor Node’s internal ESM loader to move its exception on unknown file types from within `resolve` (on detection of unknown extensions) to within `load` (if the resolved extension has no defined translator). https://github.com/nodejs/node/pull/37468

- [x] Implement chaining as described in the [design](doc/design/proposal-chaining-middleware.md), where the `default<hookName>` becomes `next` and references the next registered hook in the chain. https://github.com/nodejs/node/pull/42623

- [x] Have loaders apply to subsequent loaders. https://github.com/nodejs/loaders/blob/main/doc/design/proposal-ambient-loaders.md, https://github.com/nodejs/node/pull/43772

- [x] Move loaders off thread. https://github.com/nodejs/node/issues/43658, https://github.com/nodejs/node/pull/44710

### Milestone 2: Stability

- [x] Provide a way to register loaders from application code, such as `import { register } from 'node:module'`. https://github.com/nodejs/node/pull/46826, https://github.com/nodejs/node/pull/48559.

- [x] Replace `globalPreload` hook with new `initialize` hook; update `register` to preserve the communications channel from that hook so that we continue to provide a way to communicate between loaders code and application code. See https://github.com/nodejs/loaders/discussions/124#discussioncomment-5735397 and https://github.com/nodejs/loaders/issues/147. https://github.com/nodejs/node/pull/48842.

- [x] Remove `globalPreload` hook. https://github.com/nodejs/node/pull/49144.

- [x] Support loading source when the return value of `load` has `format: 'commonjs'`. See https://github.com/nodejs/node/issues/34753#issuecomment-735921348 and https://github.com/nodejs/loaders-test/blob/835506a638c6002c1b2d42ab7137db3e7eda53fa/coffeescript-loader/loader.js#L45-L50. https://github.com/nodejs/node/pull/47999.

- [x] Unflag `import.meta.resolve`. https://github.com/nodejs/node/pull/49028.

### Milestone 3: Usability improvements

- [x] Provide a way to register loaders via configuration, for example via adding support for `.env` files to Node.js or having `node` read configuration from a new field in `package.json` or other configuration file. See https://github.com/nodejs/node/pull/46826, [#98](https://github.com/nodejs/loaders/issues/98), https://github.com/nodejs/node/pull/43973#issuecomment-1249549346. https://github.com/nodejs/node/pull/48890.

- [ ] Integrated support for external formats.

   - [ ] Phase 1: Support identifying external modules (eg `typescript`); see https://github.com/nodejs/node/pull/49704.
   - [ ] Phase 2: Support guided remediation via package manager search (eg `npm search … typescript`).
   - [ ] Phase 3: Automatically configure Node.

- [ ] First-class support for [import maps](https://github.com/WICG/import-maps) that doesn’t require a custom loader.

- [ ] Add helper/utility functions to reduce boilerplate in user-defined hooks.

   - [ ] Start with helpers for retrieving the closest parent `package.json` associated with a specifier string; and for retrieving the `package.json` for a particular package by name (which is not necessarily the same result).

   - [ ] Potentially include all the functions that make up the ESM resolution algorithm as defined in the [spec](https://nodejs.org/api/esm.html#resolver-algorithm-specification). Create helper functions for each of the functions defined in that psuedocode: `esmResolve`, `packageImportsResolve`, `packageResolve`, `esmFileFormat`, `packageSelfResolve`, `readPackageJson`, `packageExportsResolve`, `lookupPackageScope`, `packageTargetResolve`, `packageImportsExportsResolve`, `patternKeyCompare`. (Not necessarily all with these exact names, but corresponding to these functions from the spec.)

   - [ ] Follow up with similar helper functions that make up what happens within Node’s internal `load`. (Definitions to come.)

- [ ] Helper/utility functions to allow access to the CommonJS named exports discovery algorithm (`cjs-module-lexer`).

- [ ] Hooks for customizing the REPL, including transpilation and tab completion. Support users pasting TypeScript (or CoffeeScript or whatever) into the REPL and having just as good an experience as with plain JavaScript.

   - [ ] Support top-level `await` in the REPL API, if possible.
     
   - [ ] Add support for `--eval` (or `--print`) CLI flags, extending over to support runtime `eval()` as well.

- [ ] Hooks for customizing the stack trace (in other words, a hook version of `Error.prepareStackTrace`). This would allow transpiled languages to improve the output.

- [ ] Hooks for customizing filesystem calls, for allowing things like virtual filesystems or archives treated as volumes.

- [ ] Inherit configuration blob to worker threads and child processes.
