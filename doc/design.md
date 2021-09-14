# Loaders Design

## Hooks

There are currently [three loader hooks](https://nodejs.org/api/esm.html#esm_hooks):

- `resolve`: Take a specifier (the string after `from` in an `import` statement) and convert it into an URL to be loaded.

- `load`: Take the resolved URL and return runnable code (JavaScript, Wasm, etc.) as well as the name of one of Node’s ESM loader’s [“translators”](https://github.com/nodejs/node/blob/master/lib/internal/modules/esm/translators.js): `commonjs`, `module`, `builtin` (a Node internal module like `fs`), `json` (with `--experimental-json-modules`) or `wasm` (with `--experimental-wasm-modules`).

- `globalPreload`: Define a string of JavaScript to be injected into the application global scope.

## Chaining `resolve` hooks

Say you had a chain of three loaders, `unpkg`, `http-to-https`, `cache-buster`:

1. The `unpkg` loader resolves a specifier `foo` to an URL `http://unpkg.com/foo`.

2. The `http-to-https` loader rewrites that URL to `https://unpkg.com/foo`.

3. The `cache-buster` that takes the URL and adds a timestamp to the end, so like `https://unpkg.com/foo?ts=1234567890`.

In the new loaders design, these three loaders could be implemented as follows:

### `unpkg` loader

```js
export async function resolve(specifier, context, next) { // next is Node’s resolve
  if (isBareSpecifier(specifier)) {
    return `http://unpkg.com/${specifier}`;
  }
  return next(specifier, context);
}
```

### `http-to-https` loader

```js
export async function resolve(specifier, context, next) { // next is the unpkg loader’s resolve
  const result = await next(specifier, context);
  if (result.url.startsWith('http://')) {
    result.url = `https${result.url.slice('http'.length)}`;
  }
  return result;
}
```

### `cache-buster` loader

```js
export async function resolve(specifier, context, next) { // next is the http-to-https loader’s resolve
  const result = await next(specifier, context);
  if (supportsQueryString(result.url)) { // exclude data: & friends
    // TODO: do this properly in case the URL already has a query string
    result.url += `?ts=${Date.now()}`;
  }
  return result;
}
```

The hook functions nest: each one always just returns a string, like Node’s `resolve`, and the chaining happens as a result of calling `next`; and if a hook doesn’t call `next`, the chain short-circuits. The API would be `node --loader unpkg --loader http-to-https --loader cache-buster`, following the pattern set by `--require`.

## Chaining `load` hooks

Chaining `load` hooks would be similar to chaining `resolve` hooks, though slightly more complicated in that instead of returning a single string, each `load` hook returns an object `{ format, source }` where `source` is the loaded module’s source code/contents and `format` is the name of one of Node’s ESM loader’s [“translators”](https://github.com/nodejs/node/blob/master/lib/internal/modules/esm/translators.js): `commonjs`, `module`, `builtin` (a Node internal module like `fs`), `json` (with `--experimental-json-modules`) or `wasm` (with `--experimental-wasm-modules`).

Currently, Node’s internal ESM loader throws an error on unknown file types: `import('file.javascript')` throws, even if the contents of `file.javascript` are perfectly acceptable JavaScript. This error happens during Node’s internal `resolve` when it encounters a file extension it doesn’t recognize; hence the current [CoffeeScript loader example](https://nodejs.org/api/esm.html#esm_transpiler_loader) has lots of code to tell Node to allow CoffeeScript file extensions. We should move this validation check to be after the format is determined, which is one of the return values of `load`; so basically, it’s the responsibility of `load` to return a `format` that Node recognizes. Node’s internal `load` doesn’t know to resolve a URL ending in `.coffee` to `module`, so Node would continue to error like it does now; but the CoffeeScript loader under this new design no longer needs to hook into `resolve` at all, since it can determine the format of CoffeeScript files within `load`. In code:

### `coffeescript` loader

```js
import CoffeeScript from 'coffeescript';

// CoffeeScript files end in .coffee, .litcoffee or .coffee.md
const extensionsRegex = /\.coffee$|\.litcoffee$|\.coffee\.md$/;

export async function load(url, context, next) {
  const result = await next(url, context);

  // The first check is technically not needed but ensures that
  // we don’t try to compile things that already _are_ compiled.
  if (result.format === undefined && extensionsRegex.test(url)) {
    // For simplicity, all CoffeeScript URLs are ES modules.
    const format = 'module';
    const source = CoffeeScript.compile(result.source, { bare: true });
    return {format, source};
  }
  return result;
}
```

And the other example loader in the docs, to allow `import` of `https://` URLs, would similarly only need a `load` hook:

### `https` loader

```js
import { get } from 'https';

export async function load(url, context, next) {
  if (url.startsWith('https://')) {
    let format; // default: format is undefined
    const source = await new Promise((resolve, reject) => {
      get(url, (res) => {
        // Determine the format from the MIME type of the response
        switch (res.headers['content-type']) {
          case 'application/javascript':
          case 'text/javascript': // etc.
            format = 'module';
            break;
          case 'application/node':
          case 'application/vnd.node.node':
            format = 'commonjs';
            break;
          case 'application/json':
            format = 'json';
            break;
          // etc.
        }

        let data = '';
        res.on('data', (chunk) => data += chunk);
        res.on('end', () => resolve({ source: data }));
      }).on('error', (err) => reject(err));
    });
    return {format, source};
  }

  return next(url, context);
}
```

If these two loaders are used together, where the `coffeescript` loader’s `next` is the `https` loader’s hook and `https` loader’s `next` is Node’s native hook, then for a URL like `https://example.com/module.coffee`:

1. The `https` loader would load the source over the network, but return `format: undefined`, assuming the server supplied a correct `Content-Type` header like `application/vnd.coffeescript` which our `https` loader doesn’t recognize.

2. The `coffeescript` loader would get that `{ source, format: undefined }` early on from its call to `next`, and set `format: 'module'` based on the `.coffee` at the end of the URL. It would also transpile the source into JavaScript. It then returns `{ format: 'module', source }` where `source` is runnable JavaScript rather than the original CoffeeScript.

## Chaining `globalPreload` hooks

For now, we think that this wouldn’t be chained the way `resolve` and `load` would be. This hook would just be called sequentially for each registered loader, in the same order as the loaders themselves are registered. If this is insufficient, for example for instrumentation use cases, we can discuss and potentially change this to follow the chaining style of `load`.
