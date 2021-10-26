# Chaining Hooks “Iterative” Design

## Chaining `resolve` hooks

Say you had a chain of three loaders:

1. `unpkg` resolves a specifier `foo` to an URL `http://unpkg.com/foo`.
2. `https` rewrites that URL to `https://unpkg.com/foo`.
3. `cache-buster` takes the URL and adds a timestamp to the end, like `https://unpkg.com/foo?ts=1234567890`.

```console
node \
--loader unpkg \
--loader https \
--loader cache-buster
```

These would be called in the following sequence:

`unpkg` → `https` → `cache-buster`

Resolve hooks would have the following signature:

```ts
export async function resolve(
  interimResult: {             // results from the previous hook
    format = '',
    url = '',
  },
  context: {
    conditions = string[],     // export conditions of the relevant package.json
    parentUrl = null,          // foo.mjs imports bar.mjs
                               // when module is bar.mjs, parentUrl is foo.mjs
    originalSpecifier: string, // the original value of the import specifier
  },
  defaultResolve,              // node's default resolve hook
): {
  format?: string,             // a hint to the load hook (it can be ignored)
  signals?: {                  // signals from this hook to the ESMLoader
    contextOverride?: object,  // the new `context` argument for the next hook
    interimIgnored?: true,     // interimResult was intentionally ignored
    shortCircuit?: true,       // `resolve` chain should be terminated
  },
  url: string,                 // the final hook must return a valid URL string
} {
```

A hook including `shortCircuit: true` will cause the chain to short-circuit, immediately terminating the hook's chain (no subsequent `resolve` hooks are called).

### `unpkg` resolver

<details>
<summary>`unpkg-resolver.mjs`</summary>

```js
export async function resolve(
  interimResult,
  { originalSpecifier },
) {
  if (isBareSpecifier(originalSpecifier)) return `http://unpkg.com/${originalSpecifier}`;
}
```
</details>

### `https` resolver

<details>
<summary>`https-loader.mjs`</summary>

```js
export async function resolve(
  interimResult,
  context,
) {
  const url = new URL(interimResult.url); // this can throw, so handle appropriately

  if (url.protocol = 'http:') url.protocol = 'https:';

  return { url: url.toString() };
}

export async function load(/* … */) {/* … */ }
```
</details>

### `cache-buster` resolver

<details>
<summary>`cache-buster-resolver.mjs`</summary>

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

* `babel` backwards time-machine
* `coffeescript` transforms coffeescript to vanilla javascript
* `https` loads source from remote

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
    conditions = string[],     // export conditions of the relevant package.json
    parentUrl = null,          // foo.mjs imports bar.mjs
                               // when module is bar, parentUrl is foo.mjs
    resolvedUrl: string,       // url to which the resolve hook chain settled
  },
  defaultLoad: function,       // node's default load hook
): {
  format: string,              // the final hook must return any node supports
  signals?: {                  // signals from this hook to the ESMLoader
    contextOverride?: object,  // the new `context` argument for the next hook
    interimIgnored?: true,     // interimResult was intentionally ignored
    shortCircuit?: true,       // `resolve` chain should be terminated
  },
  source: string | ArrayBuffer | TypedArray,
} {
```

A hook including `shortCircuit: true` will cause the chain to short-circuit, immediately terminating the hook's chain (no subsequent `load` hooks are called).

### `https` loader

<details>
<summary>`https-loader.mjs`</summary>

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
<summary>`coffeescript-loader.mjs`</summary>

```js
export async function resolve(/* … */) {/* … */ }

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
<summary>`babel-loader.mjs`</summary>

```js
export async function resolve(/* … */) {/* … */ }

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
