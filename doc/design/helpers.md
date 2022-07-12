# Helpers

A user loader that defines a hook might need to reimplement all of Node’s original version of that hook, if the user hook can’t call `next` to get the result of the internal version. A case where this might occur is input that Node would error on, for example a `resolve` hook trying to resolve a protocol that Node doesn’t support. For such a hook, it could involve a lot of boilerplate or dependencies to reimplement all of the logic contained within Node’s internal version of that hook. We plan to create helper functions to reduce that need.

These will be added in stages, starting with helpers for the `resolve` hook that cover the various steps that Node’s internal `resolve` performs.

## Usage

The intended usage for these helpers is to eliminate boilerplate within user-defined hooks. For example:

```js
import { isBareSpecifier, packageResolve } from 'node:module';
import { readFileSync } from 'node:fs';
import { join } from 'node:path';
import { fileURLToPath, pathToFileURL } from 'node:url';


const needsTranspilation = new Set();

export function resolve(specifier, context, next) {
  if (!isBareSpecifier(specifier)) {
    return next(specifier, context);
  }

  const pathToPackage = fileURLToPath(packageResolve(specifier, context.parentURL, context.conditions));
  const pathToPackageJson = join(pathToPackage, 'package.json');
  const packageMetadata = JSON.parse(readFileSync(pathToPackageJson, 'utf-8'));

  if (!packageMetadata.exports && packageMetadata.module) {
    // If this package has a "module" field but no "exports" field,
    // return the value of "module" and transpile later
    // within a `load` hook
    return {
      url: pathToFileURL(join(pathToPackage, packageMetadata.module))
    }
  }
}

export async function load(url, context, next) {
  if (!needsTranspilation.has(url)) {
    return next(url, context);
  }

  // TODO: Transpile the faux-ESM in the "module" URL
  // and return the transpiled, runnable ESM source
}
```

## `resolve` helpers

### `packageResolve`

Public reference to https://github.com/nodejs/node/blob/3350d9610864af3219de7dd20e3ac18b3c214c52/lib/internal/modules/esm/resolve.js#L847-L910.

Existing signature:

```js
/**
 * @param {string} specifier
 * @param {string | URL | undefined} base
 * @param {Set<string>} conditions
 * @returns {resolved: URL, format? : string}
 */
function packageResolve(specifier, base, conditions) {
```

New signature, where many supporting functions that this function calls are passed in (and if left undefined, the default versions are used):

```js
function packageResolve(specifier, base, conditions, {
  parsePackageName,
  getPackageScopeConfig,
  packageExportsResolve,
  findPackageJson, // Part of current packageResolve extracted into its own function
  getPackageConfig,
  legacyMainResolve,
}) {
```

The middle of the function, where it walks up the disk looking for `package.json`, would be moved into a separate function `findPackageJson` that would follow a pattern similar to this, where the various file system-related functions could all be overridden (so that a virtual filesystem could be simulated, for example).

### `findPackageJson`

Extracted from `packageResolve` https://github.com/nodejs/node/blob/3350d9610864af3219de7dd20e3ac18b3c214c52/lib/internal/modules/esm/resolve.js#L873-L887 plus the `while` condition:

```js
function findPackageJson(packageName, base, isScoped, {
  fileURLToPath,
  tryStatSync,
})
```
