# Chaining Hooks “Iterative” Design

## Chaining `resolve` hooks

Say you had a chain of three loaders:

1. `unpkg` resolves a specifier `foo` to an URL `http://unpkg.com/foo`.
2. `http-to-https` rewrites that URL to `https://unpkg.com/foo`.
3. `cache-buster` takes the URL and adds a timestamp to the end, like `https://unpkg.com/foo?ts=1234567890`.

Following the pattern of `--require`:

```console
node \
  --loader unpkg \
  --loader http-to-https \
  --loader cache-buster
```

These would be called in the following sequence:

`unpkg` → `http-to-https` → `cache-buster`

Resolve hooks would have the following signature:

```ts
export async function resolve(
  interimResult: {             // results from the previous hook
    format = '',
    url = '',
  },
  context: {
    conditions = string[],     // Export conditions of the relevant package.json
    parentUrl = null,          // The module importing this one, or null if
                               // this is the Node entry point
    specifier: string,         // The original value of the import specifier
  },
  defaultResolve,              // Node's default resolve hook
): {
  format?: string,             // A hint to the load hook (it might be ignored)
  signals?: {                  // Signals from this hook to the ESMLoader
    contextOverride?: object,  // A new `context` argument for the next hook
    interimIgnored?: true,     // interimResult was intentionally ignored
    shortCircuit?: true,       // `resolve` chain should be terminated
  },
  url: string,                 // The absolute URL that this input resolves to
} {
```

A hook including `shortCircuit: true` will cause the chain to short-circuit, immediately terminating the hook's chain (no subsequent `resolve` hooks are called).

### `unpkg` loader

<details>
<summary>`unpkg.mjs`</summary>

```js
export async function resolve(
  interimResult,
  { originalSpecifier },
) {
  if (isBareSpecifier(originalSpecifier)) return `http://unpkg.com/${originalSpecifier}`;
}
```
</details>

### `http-to-https` loader

<details>
<summary>`http-to-https.mjs`</summary>

```js
export async function resolve(
  interimResult,
  context,
) {
  const url = new URL(interimResult.url); // this can throw, so handle appropriately

  if (url.protocol = 'http:') url.protocol = 'https:';

  return { url: url.toString() };
}
```
</details>

### `cache-buster` resolver

<details>
<summary>`cache-buster.mjs`</summary>

```js
export async function resolve(
  interimResult,
) {
  const url = new URL(interimResult.url); // this can throw, so handle appropriately

  if (supportsQueryString(url.protocol)) { // exclude data: & friends
    url.searchParams.set('t', Date.now());
  }

  return { url: url.toString() };
}

function supportsQueryString(/* … */) {/* … */}
```
</details>


## Chaining `load` hooks

Say you had a chain of three loaders:

* `babel` transforms modern JavaScript source into a specified target
* `coffeescript` transforms CoffeeScript source into JavaScript source
* `https` fetches `https:` URLs and returns their contents

Following the pattern of `--require`:

```console
node \
  --loader https \
  --loader babel \
  --loader coffeescript \
```

These would be called in the following sequence:

(`https` OR `defaultLoad`) → `coffeescript` → `babel`

1. `defaultLoad` / `https` needs to be first to actually get the source, which is fed to the subsequent loader
1. `coffeescript` receives the raw source from the previous loader and transpiles coffeescript files to regular javascript
1. `babel` receives potentially bleeding-edge JavaScript and transforms it to some ancient JavaScript target

The below examples are not exhaustive and provide only the gist of what each loader needs to do and how it interacts with the others.

Load hooks would have the following signature:

```ts
export async function load(
  interimResult: {             // result from the previous hook
    format = '',               // the value if `resolve` settled with a `format`
                               // until a load hook provides a different value
    source = '',
  },
  context: {
    conditions = string[],     // Export conditions of the relevant package.json
    parentUrl = null,          // The module importing this one, or null if
                               // this is the Node entry point
    resolvedUrl: string,       // The URL returned by the last hook of the
                               // `resolve` chain
  },
  defaultLoad: function,       // Node's default load hook
): {
  format: 'builtin' | 'commonjs' | 'module' | 'json' | 'wasm', // A format
                               // that Node understands
  signals?: {                  // Signals from this hook to the ESMLoader
    contextOverride?: object,  // A new `context` argument for the next hook
    interimIgnored?: true,     // interimResult was intentionally ignored
    shortCircuit?: true,       // `resolve` chain should be terminated
  },
  source: string | ArrayBuffer | TypedArray, // The source for Node to evaluate
} {
```

A hook including `shortCircuit: true` will cause the chain to short-circuit, immediately terminating the hook's chain (no subsequent `load` hooks are called).

### `https` loader

<details>
<summary>`https.mjs`</summary>

```js
export async function load(
  interimResult,
  { resolvedUrl },
) {
  if (interimResult.source) return; // step aside (content already retrieved)

  if (!resolvedUrl.startsWith('https://')) return; // step aside

  return new Promise(function loadHttpsSource(resolve, reject) {
    get(resolvedUrl, function getHttpsSource(rsp) {
      const format = mimeTypeToFormat.get(rsp.headers['content-type']);
      let source = '';

      rsp.on('data', (chunk) => source += chunk);
      rsp.on('end', () => resolve({ format, source }));
      rsp.on('error', reject);
    });
  });
}

const mimeTypeToFormat = new Map([
  ['application/node', 'commonjs'],
  ['application/javascript', 'module'],
  ['text/javascript', 'module'],
  ['application/json', 'json'],
  ['text/coffeescript', 'coffeescript'],
  // …
]);
```
</details>

### `coffeescript` loader

<details>
<summary>`coffeescript.mjs`</summary>

```js
export async function load(
  interimResult, // possibly output of https-loader
  context,
  defaulLoad,
) {
  const { resolvedUrl } = context;
  if (!coffeescriptExtensionsRgx.test(resolvedUrl)) return; // step aside

  const format = interimResult.format || await getPackageType(resolvedUrl);
  if (format === 'commonjs') return { format };

  const rawSource = (
    interimResult.source
    || await defaulLoad(resolvedUrl, { ...context, format }).source
  )
  const transformedSource = CoffeeScript.compile(rawSource.toString(), {
    bare: true,
    filename: resolvedUrl,
  });

  return {
    format,
    source: transformedSource,
  };
}

function getPackageType(url) {/* … */ }
const coffeescriptExtensionsRgs = /* … */
```
</details>

### `babel` loader

<details>
<summary>`babel.mjs`</summary>

```js
export async function load(
  interimResult, // possibly output of coffeescript-loader
  context,
  defaulLoad,
) {
  const { resolvedUrl } = context;
  const babelConfig = await getBabelConfig(resolvedUrl);

  const format = (
    interimResult.format
    || babelOutputToFormat.get(babelConfig.output.format)
  );

  if (format === 'commonjs') return { format };

  const sourceToTranspile = (
    interimResult.source
    || await defaulLoad(resolvedUrl, { ...context, format }).source
  );
  const transformedSource = Babel.transformSync(
    sourceToTranspile.toString(),
    babelConfig,
  ).code;

  return {
    format,
    source: transformedSource,
  };
}

function getBabelConfig(url) {/* … */ }
const babelOutputToFormat = new Map([
  ['cjs', 'commonjs'],
  ['esm', 'module'],
  // …
]);
```
</details>

## Chaining `readFile` hooks

Say you had a chain of three loaders:

* `zip` adds a virtual filesystem layer for in-zip access
* `tgz` does the same but for tgz archives
* `warc` does the same for warc archives.

Following the pattern of `--require`:

```console
node \
  --loader zip \
  --loader tgz \
  --loader warc
```

These would be called in the following sequence:

(`zip` OR `defaultReadFile`) → `tgz` → `warc`

1. `defaultReadFile` / `zip` needs to be first to know whether the manifest exists on the actual filesystem, which is fed to the subsequent loader
1. `tgz` receives the raw source from the previous loader and, if necessary, checks for the manifest existence via its own rules
1. `warc` does the same thing

LoadManifest hooks would have the following signature:

```ts
export async function readFile(
  url: string,                 // A URL pointing to a location; whether the file
                               // exists or not isn't guaranteed
  interimResult: {             // result from the previous hook
    data: string | ArrayBuffer | TypedArray | null, // The content of the
                               // file, or `null` if it doesn't exist.
  },
  context: {
    conditions = string[],     // Export conditions of the relevant package.json
  },
  defaultLoadManifest: function, // Node's default load hook
): {
  signals?: {                  // Signals from this hook to the ESMLoader
    contextOverride?: object,  // A new `context` argument for the next hook
    interimIgnored?: true,     // interimResult was intentionally ignored
    shortCircuit?: true,       // `resolve` chain should be terminated
  },
  data: string | ArrayBuffer | TypedArray | null, // The content of the
                               // file, or `null` if it doesn't exist.
} {
```
