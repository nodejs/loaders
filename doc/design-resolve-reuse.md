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

### Utility functions API

Here is an example of how the utility API could look like. It is written in typescript notation to make the types of parameters clear. The design of this API is mainly an effect of refactoring the existing functions and may have looked different if designed from scratch.

The main functions are `packageExportsResolve` and `packageImportsResolve`. Both these function make use of a function of type `PackageResolve` that is provided by the calling application. The `PackageResolve` function is also the main function to start the resolve, and then it calls into the utility functions which may call back into the `PackageResolve` function. This is becuase of the recursive nature of the resolve, eg. an export/import can be a package name that has to be resolved.

```ts
/**
 * This needs to be implemented by the caller
 */
type PackageResolve = (
  specifier: string,
  base: string | URL | undefined,
  conditions: ReadonlySet<string>,
  isDirectory: IsDirectory,
  readFile: ReadFile
) => ReadonlyArray<URL>;

export type IsDirectory = (path: string) => boolean;
export type ReadFile = (filename: string) => string | undefined;

/**
 * Relevant parts of package.json
 */
type PackageConfig = {
  readonly pjsonPath: string;
  readonly exists: boolean;
  readonly main: string | undefined;
  readonly name: string | undefined;
  readonly type: string;
  readonly exports: unknown | undefined;
  readonly imports: unknown | undefined;
};

export function packageExportsResolve(
  packageResolve: PackageResolve,
  packageJSONUrl: URL,
  packageSubpath: string,
  packageConfig: PackageConfig,
  base: string | URL | undefined,
  conditions: ReadonlySet<string>
): { readonly resolved: URL; readonly exact: boolean };

export function packageImportsResolve(
  packageResolve: PackageResolve,
  name: string,
  base: string | undefined,
  conditions: ReadonlySet<string>,
  readFile: ReadFile
): { readonly resolved: URL; readonly exact: boolean };

export function getPackageConfig(
  readFile: ReadFile,
  path: string,
  specifier: string,
  base: string | URL | undefined
): PackageConfig;

export function getConditionsSet(
  conditions: ReadonlyArray<string>
): ReadonlySet<string>;

export function shouldBeTreatedAsRelativeOrAbsolutePath(
  specifier: string
): boolean;

export function parsePackageName(
  specifier: string,
  base: string | URL | undefined
): {
  readonly packageName: string;
  readonly packageSubpath: string;
  readonly isScoped: boolean;
};

export function legacyMainResolve(
  packageJSONUrl: string | URL,
  packageConfig: PackageConfig
): ReadonlyArray<URL>;

export function resolveSelf(
  packageResolve: PackageResolve,
  base: string | URL | undefined,
  packageName: string,
  packageSubpath: string,
  conditions: ReadonlySet<string>,
  readFile: ReadFile
): URL;

export function findPackageJson(
  packageName: string,
  base: string | URL | undefined,
  isScoped: boolean,
  isDirectory: IsDirectory
): readonly [packageJSONUrl: URL, packageJSONPath: string] | undefined;
```

### Example of using utility functions to resolve

Note that the resolve from the `packageResolve` function can be ambigous in that an array of multiple possible URLs is returned. This is becuase the `legacyMainResolve` is ambigous and is avoiding file system access by returning all possibilites rather than looking in the file system for what exists. The application is left to sort out which possibility is the right one with it's own logic.

```ts
import * as rua from "utility-functions-from-above";

function startResolve(
  specifier: string,
  base: string | undefined,
  conditionsArray: ReadonlyArray<string>,
  isDirectory: IsDirectory,
  readFile: ReadFile
): ResolveReturn | undefined {
  // Convert conditions to set
  const conditions = rua.getConditionsSet(conditionsArray);

  // Resolve path specifiers
  if (rua.shouldBeTreatedAsRelativeOrAbsolutePath(specifier)) {
    // Application specific logic to resolve path specifiers
    return appliationLogicToResolvePathSpecifiers();
  }

  // Resolve bare specifiers
  let possibleUrls: ReadonlyArray<URL> = [];
  if (specifier.startsWith("#")) {
    // Use utility function to resolve
    const { resolved } = rua.packageImportsResolve(
      packageResolve,
      specifier,
      base,
      conditions,
      readFile
    )!;
    possibleUrls = [resolved];
  } else {
    // Use application specific packageResolve() specified below
    possibleUrls = packageResolve(
      specifier,
      base,
      conditions,
      isDirectory,
      readFile
    );
  }

  // At this point the bare specifier is resolved to one or more possible files
  // Use application specific logic to determine which one to use (or return undefined if none exists)
  return applicationLogicToSortOutWhichUrlToUse();
}

/**
 * This function resolves bare specifiers that refers to packages (not node:, data: bare specifiers)
 */
function packageResolve(
  specifier: string,
  base: string | URL | undefined,
  conditions: ReadonlySet<string>,
  isDirectory: IsDirectory,
  readFile: ReadFile
): ReadonlyArray<URL> {
  // Parse the specifier as a package name (package or @org/package) and separate out the sub-path
  const { packageName, packageSubpath, isScoped } = rua.parsePackageName(
    specifier,
    base
  );

  // ResolveSelf
  // Check if the specifier resolves to the same package we are resolving from
  const selfResolved = rua.resolveSelf(
    packageResolve,
    base,
    packageName,
    packageSubpath,
    conditions,
    readFile
  );
  if (selfResolved) {
    return [selfResolved];
  }

  // Find package.json by ascending the file system
  const packageJsonMatch = rua.findPackageJson(
    packageName,
    base,
    isScoped,
    isDirectory
  );

  // If package.json was found, resolve from it's exports or main field
  if (packageJsonMatch) {
    const [packageJSONUrl, packageJSONPath] = packageJsonMatch;
    const packageConfig = rua.getPackageConfig(
      readFile,
      packageJSONPath,
      specifier,
      base
    );
    if (packageConfig.exports !== undefined && packageConfig.exports !== null) {
      const per = rua.packageExportsResolve(
        packageResolve,
        packageJSONUrl,
        packageSubpath,
        packageConfig,
        base,
        conditions
      ).resolved;
      return [per];
    }
    if (packageSubpath === ".") {
      return rua.legacyMainResolve(packageJSONUrl, packageConfig);
    }
    return [new URL(packageSubpath, packageJSONUrl)];
  }

  return [];
}
```
