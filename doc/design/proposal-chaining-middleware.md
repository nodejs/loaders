# Chaining Hooks “Middleware” Design

## Chaining `resolve` hooks

Say you had a chain of three loaders:

1. `unpkg` resolves a specifier `foo` to an URL `http://unpkg.com/foo`.
2. `http-to-https` rewrites that URL to `https://unpkg.com/foo`.
3. `cache-buster` takes the URL and adds a timestamp to the end, like `https://unpkg.com/foo?ts=1234567890`.

The hook functions nest: each one must always return a plain object, and the chaining happens as a result of each function calling `next()`, which is a reference to the subsequent loader’s hook.

A hook that fails to return triggers an exception. A hook that returns without calling `next()`, and without returning `shortCircuit: true`, also triggers an exception. These errors are to help prevent unintentional breaks in the chain.

Following the pattern of `--require`:

```console
node \
  --loader unpkg \
  --loader http-to-https \
  --loader cache-buster
```

These would be called in the following sequence: `cache-buster` calls `http-to-https`, which calls `unpkg`. Or in JavaScript terms, `cacheBuster(httpToHttps(unpkg(input)))`.

Resolve hooks would have the following signature:

```ts
export async function resolve(
  specifier: string,         // The original specifier
  context: {
    conditions = string[],   // Export conditions of the relevant `package.json`
    parentUrl = null,        // The module importing this one, or null if
                             // this is the Node entry point
  },
  next: function,            // The subsequent `resolve` hook in the chain,
                             // or Node’s default `resolve` hook after the
                             // last user-supplied `resolve` hook
): {
  format?: string,           // A hint to the load hook (it might be ignored)
  shortCircuit?: true,       // A signal that this hook intends to terminate
                             // the chain of `resolve` hooks
  url: string,               // The absolute URL that this input resolves to
} {
```

### `cache-buster` loader

<details>
<summary>`cache-buster.mjs`</summary>

```js
export async function resolve(
  specifier,
  context,
  next, // In this example, `next` is https’ resolve
) {
  const result = await next(specifier, context);

  const url = new URL(result.url);

  if (url.protocol !== 'data:')) { // `data:` URLs don’t support query strings
    url.searchParams.set('ts', Date.now());
  }

  return { url: url.href };
}
```
</details>

### `http-to-https` loader

<details>
<summary>`http-to-https.mjs`</summary>

```js
export async function resolve(
  specifier,
  context,
  next, // In this example, `next` is unpkg’s resolve
) {
  const result = await next(specifier, context);

  const url = new URL(result.url);

  if (url.protocol === 'http:') {
    url.protocol = 'https:';
  }

  return { url: url.href };
}
```
</details>

### `unpkg` loader

<details>
<summary>`unpkg.mjs`</summary>

```js
export async function resolve(
  specifier,
  context,
  next, // In this example, `next` is Node’s default `resolve`
) {
  if (isBareSpecifier(specifier)) { // Implemented elsewhere
    return { url: `http://unpkg.com/${specifier}` };
  }

  return next(specifier, context);
}
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
  --loader babel \
  --loader coffeescript \
  --loader https
```

These would be called in the following sequence: `babel` calls `coffeescript`, which calls `https`. Or in JavaScript terms, `babel(coffeescript(https(input)))`:

Load hooks would have the following signature:

```ts
export async function load(
  resolvedUrl: string,       // The URL returned by the last hook of the
                             // `resolve` chain
  context: {
    conditions = string[],   // Export conditions of the relevant `package.json`
    parentUrl = null,        // The module importing this one, or null if
                             // this is the Node entry point
    resolvedFormat?: string, // The format returned by the last hook of the
                             // `resolve` chain
  },
  next: function,            // The subsequent `load` hook in the chain,
                             // or Node’s default `load` hook after the
                             // last user-supplied `load` hook
): {
  format: 'builtin' | 'commonjs' | 'module' | 'json' | 'wasm', // A format
                             // that Node understands
  shortCircuit?: true,       // A signal that this hook intends to terminate
                             // the chain of `load` hooks
  source: string | ArrayBuffer | TypedArray, // The source for Node to evaluate
} {
```

### `babel` loader

<details>
<summary>`babel.mjs`</summary>

```js
const babelOutputToFormat = new Map([
  ['cjs', 'commonjs'],
  ['esm', 'module'],
  // …
]);

export async function load(
  url,
  context,
  next, // In this example, `next` is coffeescript’s hook
) {
  const babelConfig = await getBabelConfig(url); // Implemented elsewhere

  const format = babelOutputToFormat.get(babelConfig.output.format);

  if (format === 'commonjs') {
    return { format, source: '' }; // Source is ignored for CommonJS
  }

  const { source: transpiledSource } = await next(url, { ...context, format });
  const { code: transformedSource } = Babel.transformSync(transpiledSource.toString(), babelConfig);

  return { format, source: transformedSource };
}
```
</details>

### `coffeescript` loader

<details>
<summary>`coffeescript.mjs`</summary>

```js
// CoffeeScript files end in .coffee, .litcoffee or .coffee.md.
const extensionsRegex = /\.coffee$|\.litcoffee$|\.coffee\.md$/;

export async function load(
  url,
  context,
  next, // In this example, `next` is https’ hook
) {
  if (!extensionsRegex.test(url)) { // Skip this hook for non-CoffeeScript imports
    return next(url, context);
  }

  const format = await getPackageType(url); // Implemented elsewhere

  if (format === 'commonjs') {
    return { format, source: '' }; // Source is ignored for CommonJS
  }

  const { source: rawSource } = await next(url, { ...context, format });
  const transformedSource = CoffeeScript.compile(rawSource.toString(), {
    bare: true,
    filename: url,
  });

  return { format, source: transformedSource };
}
```
</details>

### `https` loader

<details>
<summary>`https.mjs`</summary>

```js
import { get } from 'node:https';

const mimeTypeToFormat = new Map([
  ['application/node', 'commonjs'],
  ['application/javascript', 'module'],
  ['text/javascript', 'module'],
  ['application/json', 'json'],
  ['application/wasm', 'wasm'],
  ['text/coffeescript', 'coffeescript'],
  // …
]);

export async function load(
  url,
  context,
  next, // In this example, `next` is Node’s default `load`
) {
  if (!url.startsWith('https://')) { // Skip this hook for non-https imports
    return next(url, context);
  }

  return new Promise(function loadHttpsSource(resolve, reject) {
    get(url, function getHttpsSource(res) {
      const format = mimeTypeToFormat.get(res.headers['content-type']);
      let source = '';
      res.on('data', (chunk) => source += chunk);
      res.on('end', () => resolve({ format, source }));
      res.on('error', reject);
    }).on('error', (err) => reject(err));
  });
}
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

These would be called in the following sequence: `zip` calls `tgz`, which calls `warc`. Or in JavaScript terms, `zip(tgz(warc(input)))`:

Load hooks would have the following signature:

```ts
export async function readFile(
  url: string,               // A URL pointing to a location; whether the file
                             // exists or not isn't guaranteed
  context: {
    conditions = string[],   // Export conditions of the relevant `package.json`
  },
  next: function,            // The subsequent `readFile` hook in the chain,
                             // or Node’s default `readFile` hook after the
                             // last user-supplied `readFile` hook
): {
  data: string | ArrayBuffer | TypedArray | null, // The content of the
                             // file, or `null` if it doesn't exist.
  shortCircuit?: true,       // A signal that this hook intends to terminate
                             // the chain of `load` hooks
} {
```
