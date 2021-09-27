# Chaining Hooks “Iterative” Design

## Chaining `resolve` hooks

Say you had a chain of three loaders:

* `unpkg-resolver` converts a bare specifier like `foo` to a url `http://unpkg.com/foo`.
1. `http-to-https-resolver` rewrites insecure http urls to the https, like `https://unpkg.com/foo`.
1. `cache-buster-resolver` adds a timestamp to a url end, like `https://unpkg.com/foo?t=1234567890`.

```console
node \
--loader unpkg-resolver \
--loader https-resolver \
--loader cache-buster-resolver
```

These would be called in the following sequence:

`unpkg-resolver` → `https-resolver` → `cache-buster-resolver`

Resolve hooks have the following signature:

```js
export async function resolve(
	interimResult: { // The result from the previous hook
	  format = '',   // if resolve settled with a `format`, this is that value
	                 // until a load hook provides a different value
	  url = '',      // the most recently provided value from a previous hook
	},
	context: {
	  conditions,    // export conditions (from the relevant package.json)
	  parentUrl,     // foo.mjs imports bar.mjs
	                 // when module is bar, parentUrl is foo.mjs
		originalSpecifier, // The original value of the import specifier
	},
	defaultResolve,  // node's default resolve hook
	shortCircuit,    // special function for terminating the chain early
): { format: string, url: string } {
```

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
<summary>`https-resolver.mjs`</summary>

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
<summary>`cachebuster-resolver.mjs`</summary>

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
```
</details>


## Chaining `load` hooks

Say you had a chain of three loaders:

* `babel-loader` backwards time-machine
* `coffeescript-loader` transforms coffeescript to vanilla javascript
* `https-loader` loads source from remote

```console
node \
--loader https-loader \
--loader babel-loader \
--loader coffeescript-loader \
```

These would be called in the following sequence:

(`https-loader` OR `defaultLoad`) → `coffeescript-loader` → `babel-loader`

1. `defaultLoad` / `https-loader` needs to be first to actually get the source, which is fed to the subsequent loader
1. `coffeescript-loader` receives the raw source from the previous loader and transpiles coffeescript files to regular javascript
1. `babel-loader` receives potentially bleeding-edge JavaScript and transforms it to some ancient JavaScript target

The below examples are not exhaustive and provide only the gist of what each loader needs to do and how it interacts with the others.

Load hooks have the following signature:

```js
export async function load(
	interimResult: { // The result from the previous hook
	  format = '',   // if resolve settled with a `format`, this is that value
	                 // until a load hook provides a different value
	  source = '',   // 
	},
	context: {
	  conditions,    // export conditions (from the relevant package.json)
	  parentUrl,     // foo.mjs imports bar.mjs
	                 // when module is bar, parentUrl is foo.mjs
	  resolvedUrl,   // the url to which the resolve hook chain settled
	},
	defaultLoad,     // node's default load hook
	shortCircuit,    // special function for terminating the chain early
): { format: string, source: string | ArrayBuffer | TypedArray } {
```

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