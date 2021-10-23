# Re-use of node's esm resolve algorithm

Some loaders want to run basically the same algorithm that node is doing by default (`defaultResolve`) but with a few tweaks. For example these loaders would benefit from re-use of `defaultResolve`:

- Yarn PnP loader that read files from compressed archives instead of the file system
- Typescript loader that translate the path to .ts/.tsx files

In many cases it comes down to just altering how filsystem access is handled. For Yarn PnP it would just want to change where files are read, but the resolution algorithm would be the same as the default.

Typescript files import from the future output so the imported file may not exist but need to be mapped back to the source file which potentially is in another directory (eg. if using project references and yarn workspaces).

In other cases where larger alterations of the algorithm is desired it still might be useful to call into parts of the default algorithm(?).

# Analysis of current file system access in defaultResolve

The default resolve algorithm is implemented in [resolve.js](https://github.com/nodejs/node/blob/master/lib/internal/modules/esm/resolve.js).

It uses filesystem by direct imports of `realpathSync` and `statSync` and also indirectly by import of `packageJsonReader`.

`realpathSync` is only called from the top-level function `defaultResolve`.

The following tree shows where `statSync` and `packageJsonReader.read` are called. It's not going all the way to the top of `defaultResolve` function but stopping at functions `packageResolve`, `finalizeResolution` and `moduleResolve`.

- statSync

  - tryStatSync
    - finalizeResolution
    - packageResolve
  - fileExists
    - legacyMainResolve
      - packageResolve
    - resolveExtensionsWithTryExactName
      - resolveDirectoryEntry
        - finalizeResolution
      - finalizeResolution
    - resolveExtensions
      - resolveExtensionsWithTryExactName
        - resolveDirectoryEntry
          - finalizeResolution
        - finalizeResolution
      - resolveDirectoryEntry
        - finalizeResolution
    - resolveDirectoryEntry
      - finalizeResolution

- packageJsonReader.read

  - getPackageConfig

    - getPackageScopeConfig
      - packageImportsResolve
        - moduleResolve
          - defaultResolve
      - getPackageType (not called within resolve.js)
      - packageResolve
    - packageResolve

  - resolveDirectoryEntry
    - finalizeResolution

The analysis shows that filesystem access occurs mainly as an effect of calls to `packageResolve`, `finalizeResolution`, and `moduleResolve`.

`moduleResolve` is only called by the top-level `defaultResolve` function.
`finalizeResolution` is only called from `moduleResolve`.

`packageResolve` is called in multiple places, it is also called recursively:

- packageResolve

  - resolvePackageTargetString

    - resolvePackageTarget
      - resolvePackageTarget (recursive)
      - packageExportsResolve
        - packageResolve (recursive)
      - packageImportsResolve
        - moduleResolve
          - defaultResolve

  - moduleResolve
    - defaultResolve

# Stragegies for re-use of defaultResolve

## Filesystem hooks

Using this strategy, the loader would provide hooks for filesystem calls, specifically hooks corresponding to `realpathSync`, `statSync` and `packageJsonReader.read`.

## Utility functions

This strategy would export utility functions so a custom loader could pick which parts of the default algorithm's logic it wants to call. If utility functions are exported, they probably should not do any filesystem access or throw exceptions (but instead return a value indicating success/fail).

Specifically we could refactor `resolve.js` so that `packageExportsResolve` and `packageImportsResolve` are exported and take a custom `packageResolve` function instead of calling the internal one. The loader could then call `packageExportsResolve` and `packageImportsResolve` as utility functions while providing its own `packageResolve` so it can control filesystem access.

If the loader should provide its own `packageResolve` it could be useful to break out some parts of the default implementation. Eg. the part that finds package.json by ascending the file system and the part that checks for self resolve within the current package. The self-resolve part does not do any filesystem access so it could perhaps be moved so it does not have to be part of the `packageResolve` that the loader provides.

Using this strategy the utility functions exported would be free of filesystem side-effects and the loader would do any such effects needed itself. So this would be akin to a [functional-core-imperative-shell](https://www.destroyallsoftware.com/screencasts/catalog/functional-core-imperative-shell) stratgegy. Altough the utility functions would be "pure" only in the sense that they do no filesystem side-effects or throw exceptions. Although the utility functions cannot be made 100% pure, they probably could be made idempotent.
