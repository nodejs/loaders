# preImport Proposal

## Problem Statement

With the core resolver now synchronous only, any asynchronous resolution work
is no longer possible as the resolve function must return an immediate
resolution.

For loaders that asynchronously obtain resolution information, it would be
useful to have a new hook to enable these use cases.

## preImport Hook

The hook has the signature:

```ts
export async function preImport(specifier: string, context: {
  conditions: string[],
  topLevel: boolean,
  parentURL: string | undefined
});
```

The `preImport` hook allows for tracking and asynchronous setup work for every
top-level import operation. It has no return value, although it can return
a promise to to delay the module pipeline operations. No further hooks will
be called on that module graph until this hook resolves successfully if present.

The `preImport` hook is called for each top-level import operation by the module
loader, both for the host-called imports (ie for the main entry) and for dynamic
`import()` imports. These are distinguished by the `dynamic` context.

All `preImport` hooks for all loaders are run asynchronously in parallel, and
block any further load operations (ie resolve and load) for the module graph
being imported until they all complete successfully.

Multiple import calls to the same import specifier will re-call the hook
multiple times. The first error thrown by the `preImport` hooks will be directly
returned to the specific import operation as the load failure.

## Example

<details>
<summary>Import Map Generation</summary>

Consider an import map loader which obtains the import map for a module
asynchronously:

```ts
import { Generator } from '@jspm/generator';

// stateful host import map for current session
let importMap = { imports: {}, scopes: {} };

function isUrl (specifier) {
  try {
    new URL(specifier);
    return true;
  }
  catch {
    return false;
  }
}

const isBareSpecifier = id => !id.startsWith('./') && !id.startsWith('../') &&
    !id.startsWith('/') && !isUrl(id);

export async function preImport(specifier, { conditions, parentURL }) {
  if (!isBareSpecifier(specifier)) return;
  const generator = new Generator({
    // passing the original map extends it
    inputMap: importMap,
    baseUrl: parentURL,
    defaultProvider: 'nodemodules',
    env: conditions,
  });
  await generator.install(specifier);
  // the new map will then have the new mappings and the old mappings
  importMap = generator.getMap();
}

export function resolve(specifier, { parentURL }) {
  return {
    url: importMapResolve(importMap, specifier, parentURL),
    shortCircuit: true
  };
}
```

Internally, JSPM Generator performs fetch and module dependency analysis over
the graph using `fetch` and `es-module-lexer`.

For FS operations the load hook should be able to share the natural OS cache
anyway. For network operations if the `fetch` function is shared it should be
possible to maintain a fetch cache, alternatively the `load` hook could be
added to extend from a shared cache in these operations.
</details>
