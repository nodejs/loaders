# Strip Types Design

## Problem

TypeScript support has been for a few years at the top of the list of features that users want to see in Node.js.
Discussion has been on going for a few years [#43818](https://github.com/nodejs/node/issues/43818), and the main blocker has been the lack of a clear on long term support.
TypeScript does not follow semver, and the Node.js project has been reluctant to commit to supporting TypeScript in the long term, given that the lifespan of an LTS version is 3 years.

## Goals

- Make sure Node.js can support TypeScript with the stability guarantees that has always offered.
- Avoid breaking the ecosystem by creating a new "flavor" that only works on Node.js.
- Performance.
- Keeping it simple, provide the foundations for user to build on top.

## Initial Proposal

The proposal is to add a new flag to the Node.js CLI, `--experimental-strip-types`, which will replace inline TypeScript types with whitespace.
No type checking is performed, and types are discarded.

Example:

```typescript
interface Foo {
    bar: string;
}

throw new Error("Whitespacing");

```

Becomes:

```javascript




throw new Error("Whitespacing");

```

> Note that the missing types are replaced with whitespace, so the line numbers are preserved, so that sourcemaps are not needed.

By removing types completely, we can avoid the need to commit to supporting TypeScript in the long term, as the feature is not a full TypeScript implementation.
As in JavaScript files, file extensions are required in `import` statements and `import()` expressions.
TypeScript features that depend on settings within `tsconfig.json`, such as paths or converting module formats, are unsupported.
This will solve the long term compatibility issue, but might not be as complete as a full TypeScript implementation.

## Limitations

### `.js` extension for `.ts` files

The reason is that the compiler/bundler should be responsible to resolve the correct extension at compile time.
At runtime, the extension must be correct, not to add overhead in production when it can be solved during development.
This is an issue that should apply to all tools that execute TypeScript at runtime.

### No type checking

Type checking should be done by user land tools during development, and not at runtime.
By performing at runtime, we would be adding large overhead.

### Running TypeScript in node_modules

The proposal does not support running TypeScript files in `node_modules`.
This is to avoid encouraging package maintainers to ship TypeScript files in their packages. This has been explicitly requested by TypeScript maintainers.

### Monorepos support

Due to not running TypeScript in `node_modules`, monorepos are not supported.
This is a limitation that should be solved by user land tools.
