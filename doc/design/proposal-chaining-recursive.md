# Chaining Hooks “Recursive” Design

## Chaining `resolve` hooks

Say you had a chain of three loaders:

1. The `unpkg` loader resolves a specifier `foo` to an URL `http://unpkg.com/foo`.

2. The `http-to-https` loader rewrites that URL to `https://unpkg.com/foo`.

3. The `cache-buster` that takes the URL and adds a timestamp to the end, so like `https://unpkg.com/foo?ts=1234567890`.

The hook functions nest: each one always just returns `{ [format: string,] url: string }`, and the chaining happens as a result of calling `next()`. The chain short-circuits if a hook doesn’t call `next()`. Following the pattern of `--require`:

```console
node \
--loader unpkg-resolver \
--loader https-resolver \
--loader cache-buster-resolver
```

These would be called in the following sequence (babel-loader is called first):

`cache-buster-resolver` ← `https-resolver` ← `unpkg-resolver`

1. `cache-buster-resolver` needs the output of `https-resolver` to append the query param
1. `https-resolver` needs output of unpkg to convert it to https
1. `unpkg-resolver` returns the remote url

### `cache-buster` resolver

<details>
<summary>`cachebuster-resolver.mjs`</summary>

```js
export async function resolve(
  specifier,
  context,
  next, // https-resolver
) {
  const result = await next(specifier, context);
  
  const url = new URL(result.url); // this can throw, so handle appropriately
  
  if (supportsQueryString(url.protocol)) { // exclude data: & friends
    url.searchParams.set('ts', Date.now());
    result.url = url.href;
  }
  
  return result;
}
```
</details>

### `https` resolver

<details>
<summary>`https-resolver.mjs`</summary>

```js
export async function resolve(
  specifier,
  context,
  next, // unpkg-resolver
) {
  const result = await next(specifier, context);
  
  const url = new URL(result.url); // this can throw, so handle appropriately
  
  if (url.protocol = 'http:') {
    url.protocol = 'https:';
    result.url = url.href;
  }

  return result;
}
```
</details>

### `unpkg` resolver

<details>
<summary>`unpkg-resolver.mjs`</summary>

```js
export async function resolve(
  specifier,
  context,
  next, // Node's defaultResolve
) {
  if (isBareSpecifier(specifier)) {
    return `http://unpkg.com/${specifier}`;
  }

  return next(specifier, context);
}
```
</details>

## Chaining `load` hooks

Say you had a chain of three loaders:

* `babel-loader`
* `coffeescript-loader`
* `https-loader`

```console
node \
--loader babel-loader \
--loader coffeescript-loader \
--loader https-loader \
```

These would be called in the following sequence (babel-loader is called first):

`babel-loader` ← `coffeescript-loader` ← `https-loader` ← `defaultLoader`

1. `babel-loader` needs the output of `coffeescript-loader` to transform bleeding-edge JavaScript features to some ancient target
1. `coffeescript-loader` needs the raw source (the output of `defaultLoad` / `https-loader`) to transform coffeescript files to regular javascript
1. `defaultLoad` / `https-loader` returns the actual, raw source

The below examples are not exhaustive and provide only the gist of what each loader needs to do and how it interacts with the others.

Any loader that neglects to call `next` causes a short-circuit.

### `babel` loader

<details>
<summary>`babel-loader.mjs`</summary>

```js
export async function resolve(/* … */) {/* … */ }

export async function load(
	url,
	context,
	next, // coffeescript ← https-loader ← defaultLoader
) {
	const babelConfig = await getBabelConfig(url);

	const format = babelOutputToFormat.get(babelConfig.output.format);

	if (format === 'commonjs') return { format };

	const { source: transpiledSource } = await next(url, { ...context, format });
	const { code: transformedSource } = Babel.transformSync(transpiledSource.toString(), babelConfig);

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

### `coffeescript` loader

<details>
<summary>`coffeescript-loader.mjs`</summary>

```js
export async function resolve(/* … */) {/* … */}

export async function load(
  url,
  context,
  next, // https-loader ← defaultLoader
) {
  if (!coffeescriptExtensionsRgx.test(url)) return next(url, context, defaultLoad);
  
  const format = await getPackageType(url);
  if (format === 'commonjs') return { format };

  const { source: rawSource } = await next(url, { ...context, format });
  const transformedSource = CoffeeScript.compile(rawSource.toString(), {
    bare: true,
    filename: url,
  });

  return {
    format,
    source: transformedSource,
  };
}

function getPackageType(url) {/* … */}
const coffeescriptExtensionsRgs = /* … */
```
</details>

### `https` loader

<details>
<summary>`https-loader.mjs`</summary>

```js
import { get } from 'https';

const mimeTypeToFormat = new Map([
  ['application/node', 'commonjs'],
  ['application/javascript', 'module'],
  ['application/json', 'json'],
  // …
]);

export async function load(
  url,
  context,
  next, // defaultLoader
) {
  if (!url.startsWith('https://')) return next(url, context);
  
  return new Promise(function loadHttpsSource(resolve, reject) {
    get(url, function getHttpsSource(rsp) {
      // Determine the format from the MIME type of the response
      const format = mimeTypeToFormat.get(rsp.headers['content-type']);
      let source = '';

      rsp.on('data', (chunk) => source += chunk);
      rsp.on('end', () => resolve({ format, source }));
      rsp.on('error', reject);
    })
      .on('error', (err) => reject(err));
  });
}
```
</details>

## Mock Loader

<details>
<summary>`mock-loader.mjs`</summary>

```js
```
</details>