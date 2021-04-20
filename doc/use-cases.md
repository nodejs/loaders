# Use Cases

The use cases that we aim to achieve in a loaders implementation are:

- Instrumentation of Node.js apps containing ES module code and/or a mix of ESM and CommonJS code. This includes performance and error monitoring and code coverage, among other uses.

- Mocking ES modules for tests.

- “Hot module reload” or swapping out an already-loaded ES module with a new version during development.

- Transpilation of unsupported formats (e.g. TypeScript, CoffeeScript, etc.) into supported formats (JavaScript, Wasm); or transpilation of JavaScript with unsupported features into JavaScript that is runnable by Node.js.

- Additional resource loaders, for example to load modules from HTTPS URLs or from .zip files or from a memory cache.

- Multiple uses cases at once, for example hot module reload of a transpiled file, or instrumentation of an app with HTTPS imports.

## Improvements

Technical improvements to the loaders implementation that we want to make include:

- A way for application code to communicate with loaders code.

- A way to include loaders in an application without needing command-line flags.

- Run loaders off the main thread, so that they can intercept CommonJS `require`.