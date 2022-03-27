# Ambient Loaders Design

## Problem

Loaders let side logic be applied when resolving import statements. However, this only applies to the main application: loaders themselves aren’t currently affected by any other loaders. As a result, 3rd-party loaders cannot be injected if accessing them requires going through another loader.

For instance, imagining a loader that would allow modules to be loaded from a `node_modules.zip` file (instead of the typical folder), the following wouldn’t work:

```
node --loader zip --loader ts-node ./my-tool.mts
```

Indeed, importing `ts-node` would require the `zip` loader to contribute to the resolution (so that the final path loaded is `$PROJECT/node_modules.zip/ts-node` instead of `$PROJECT/node_modules/ts-node`, which doesn't exist), but it’s not what happens today: both `zip` and `ts-node` are resolved by the default Node resolver, preventing `ts-node` from resolving.

The reason for that is an attempt to keep loaders as isolated as possible, to allow followup improvements like reusing them across workers or scaling them up. If all loaders were influenced by prior loaders, they’d effectively coalesce into a single one, which would prevent such work. Still, the problem remains.

## Proposal

If loaders cannot generally influence each other, we could have a subset of them do. Indeed, not all loaders affect the resolution so much that they are a requirement to followup loaders. We could have two levels of loaders:

- Ambient loaders would be defined via the `--ambient-loader <module>` flag. They would be loaded sequentially and would affect the resolution of all subsequent ambient and regular loaders.

- Regular loaders would be defined via the `--loader <module>` flag. They would be loaded in parallel (at least conceptually). Because they’d only be loaded after the ambient loaders have finished evaluating, their resolution would be affected by ambient loaders.

## Simple Example

Let's imagine we have the following loaders:

- `ts`, which adds support for TypeScript files
- `my-custom-loader`, which is written in TypeScript

Without the `ts` loader, we wouldn’t be able to even load the following custom loader. But by marking it as an ambient loader, Node will make sure it’ll be taken into account when resolving (and loading) `my-custom-loader`.

The command line would end up like this:

```
node --ambient-loader ts \
     --loader my-custom-loader \
     ./path/to/script.mjs
```

## More Contrived Example

Let’s imagine we have the following loaders:

- zip, which adds zip file support to the `fs` module
- pnp, which adds support for the Plug’n’Play resolution ([more details](https://yarnpkg.com/features/pnp))
- yaml, which adds yaml file support to `import`
- coffeescript, which adds coffeescript file support to `import`

The first two are critical to a successful resolution: without `zip`, PnP wouldn't be able to load files from the zip archives. Without `pnp`, Node would look for the `yaml` and `coffeescript` loaders into a `node_modules` folder, which would fail. On the other hand, `yaml` and `coffeescript` don't depend on each other.

The command line would end up like this (in practice `--ambient-loader` would probably be passed via `NODE_OPTIONS` rather than directly on the command line):

```
node --ambient-loader zip
     --ambient-loader pnp
     --loader yaml
     --loader coffeescript
     ./path/to/script.mjs
```

When resolving `coffeescript`, we’d go into the `pnp` loader (but not `coffeescript`, which isn’t an ambient loader). The `pnp` loader itself would have been loaded after the `zip` loader ran, causing its `fs` import to be resolved to a custom module adding zip support to `fs` rather than the usual builtin module.

