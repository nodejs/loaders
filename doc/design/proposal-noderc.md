# On-disk Node.js runtime configuration (noderc)

The motivation is to provide a way for users to specify runtime configuration for a Node.js application in an on-disk file that will be discovered automatically by Node.js.

For background and early discussions about the format, etc., see https://github.com/nodejs/node/issues/53787

## Overview

### `"noderc"` in package.json

An on-disk runtime configuration should be first specified in a relevant `package.json` file using the `"noderc"` field. A `package.json` file considered relevant when it's placed in any directory from the root directory to the base directory. The base directory is either the directory where the Node.js application entry point file is (if Node.js is launched to execute a file), or the current working directory (e.g. if Node.js is launched as a REPL). An example `package.json` file looks like this:

```json
{
  "name": "my-project",
  "schema": "1.0.0",
  "noderc": "./.noderc.json"
}
```

In the initial iteration, the runtime configuration file must be in JSON format, and has an extension that ends with `.json`. We are open to support more formats identified by other extensions in the future but the details will remain to be discussed and it won't be implemented in the initial iteration.

The idiomatic way to specify this configuration would be using a file named `.noderc.json` in the same directory as the `package.json`, which tends to be in the project root directory.


### Basic of `noderc` in JSON

```json
{
  "schema": 0,
  "import": [ "amaro/register", "./monitor.ts" ]
}
```

- The `schema` field is only meant for breaking changes to the schema, so it's a single number.
  - At schema 0, there is no stability guarantee about the schema
  - When we iterate on the schema to a point to consider it stable, the schema will be set to 1
  - If the noderc is using a schema that's not the latest supported by the running Node.js version (e.g. it uses 2 but the latest schema supported by the Node.js version is 3), Node.js converts the older schema to the newer schema in the underlying implementation.
  - If the noderc is using a schema that the current Node.js version doesn't support (e.g. it's schema 3 while the Node.js version only supports up to 2), an error is thrown.
  - If the noderc is using a feature that the current Node.js version doesn't support, a warning is emitted, and the unsupported feature is ignored (or an error can be thrown).
- The other fields come from a selected list of features that can be configured using noderc.
  - The most commonly used ones should be `import` and `require`, similar to `--import` and `--require`

Compared to regular CLI flags, a structured representation of the configuration allows more granular control of the behavior, for example, it may be expanded as

```json
{
  "schema": 0,
  "import": [ { "specifier": "./monitor.js", "main-thread-only": false } ]
}
```

The general naming convention of the configuration keys are snake-cased version of corresponding CLI flags (if any) without the `--` prefix, or lower-cased version of corresponding environment variables (if any). This allows easier conversions. Exceptions include configurations that are grouped by common prefixes, for example:

```json
{
  "schema": 0,
  "test": {
    "reporter": "tap",
    "name-pattern": [ "test [1-3]", "/test [4-5]/i" ]
  }
}
```

### Escape hatches for environment variables and CLI flags

While the configuration file is intended as a structural representation for configurations that are easier to extend/reuse, we can also support escape hatches to define environment variables or CLI flags via `env`, `env-file`, `exec-args`, and possibly `v8-args` (or `js-args`, to follow Chromium):

```json
{
  "schema": 0,
  "env": {
    "FOO": "BAR"
  },
  "env-file": [ "./.env.local" ],
  "exec-args": [ "--title=test" ],
  "v8-args": [ "--max-old-space-size=100" ]
}
```

Open question: when both the escape hatches and the structural representations are specified, which one should take precedence?

### Overriding the noderc file being applied

We should provide a way for users to override the noderc file that's discoverd through `package.json` lookups, or to disable it. This can be done through either environment variables, or CLI flags, or both.

The environment variable/CLI flag can specify a registered rc file in `package.json`, where `"noderc"` contains key-value pairs instead of just a string:

```json
{
  "noderc": {
    "default": "./.noderc.json",
    "test": "./.noderc.test.json",
    "test:coverage": "./.noderc.test.coverage.json"
  }
}
```

Suppose the environment variable is called `NODE_RC`, then `NODE_RC=test` results in the resolution of `./.noderc.test.json`. If `NODE_RC` is unspecified, and the `"noderc"` field in `package.json` contains key-value pairs, the rc file pointed by `"default"` will be selected by default.

Open question: what should be the key of the default noderc? `default` might still class with potential `script` fields, which may or may not be a problem in the script -> noderc mapping we'll discuss below.

### Accompanying JS APIs

This feature should have some accompanying JS APIs to:

1. Parse a given rc file
2. Serialize a given configuration
3. Querying the current configuration applied to the process (with an option to include additional configurations added by CLI flags/environment variables/runtime APIs), and where they come from
4. Querying the current schema schema, and the features supported

IDEs and other tooling are expected to use a matching Node.js version (likely specified by the "engine" field in the applicable `package.json`) to modify the rc files.

This will be left to later iterations.

### Reusing/extending configurations

A `noderc` file can extend other `noderc` files. For example, for the main `noderc` pointed by `package.json`, it has an import that enables TypeScript loading:

```json
{
  "schema": 0,
  "import": [ "amaro/register" ]
}
```

In addition, there can be another `./.noderc.prod.json` extending it for starting the server in production with monitor on:

```json
{
  "schema": 0,
  "extends": [ "./.noderc.json" ],
  "import": [ "./monitor.ts" ]
}
```

If the configuration being extended doesn't have the same schema as the one extending it, initially an error would be thrown. Though we could also consider allowing conversions on a per-file basis and merging multiple configurations in future iterations, depending on how stable the schemas are.

Open question: how should we handle override v.s. concatenation? Should we invent a special syntax? For example, to concatenate, use `"+import"`, otherwise, use `"import"`? Or is that too cryptic and we should just do "override if it's not an array, concatenate if it's an array?"


### Mapping script entries to noderc entries

The basic idea is that task runners need to perform the mapping between the `script` chosen to be run to the matching `noderc` entires. Consider this example:

```json
{
  "noderc": {
    "dev": "./.noderc.dev.json",
    "prod": "./.noderc.prod.json",
    "test": "./.noderc.test.json",
    "test:coverage": "./.noderc.test.coverage.json"
  },
  "scripts": {
    "dev": "node --watch ./app.js",
    "prod": "node ./app.js",
    "test": "node --test",
    "test:coverage": "node --test"
  }
}
```

It's the job of the task runners to match them via the environment variable described before e.g. translating `npm_lifecycle_event=test:coverage` to `NODE_RC=test:coverage` before running the `test:coverage` target.

See the directory [./examples/noderc/multi-config](./examples/noderc/multi-config) for a sketch.

Before the task runners implement these, users can choose to translate the environment variables themselves in the `scripts` target using something like `cross-env`.

```json
{
  "noderc": {
    "dev": "./.noderc.dev.json",
    "prod": "./.noderc.prod.json",
    "test": "./.noderc.test.json",
    "test:coverage": "./.noderc.test.coverage.json"
  },
  "scripts": {
    "dev": "cross-env NODE_RC=dev node --watch ./app.js",
    "prod": "cross-env NODE_RC=prod node ./app.js",
    "test": "cross-env NODE_RC=test node --test",
    "test:coverage": "cross-env NODE_RC=test:coverage  node --run test"
  }
}
```
