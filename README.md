# NodeJS Module Resolver Proposal - package.json "format" property

This proposal builds directly on the original [In Defense of Dot JS](https://github.com/dherman/defense-of-dot-js/blob/master/proposal.md) that specifies a `module` property in the package.json as a way of whitelisting modules that use ES modules syntax, as well as other discussions on package.json [properties](https://github.com/nodejs/node/wiki/ES6-Module-Detection-in-Node#option-4-meta-in-packagejson) and [flags](https://github.com/dherman/defense-of-dot-js/issues/10).

Like that specification, this specification adds a single new package.json property, but with the simplicity of only two possible values:

```json
{
  "format": "esm"
}
```

for ECMAScript Modules, and

```json
{
  "format": "cjs"
}
```

for CommonJS modules.

When loading any module, the package.json is checked using a small extension to the existing NodeJS resolution algorithm, and the resultant format is applied as the default when loading any modules from that package.

A draft specification of the NodeJS module resolution algorithm with this adjustment is included at [spec.md](spec.md).

### Loading modules without a package.json

When starting a new NodeJS project, the package.json would need this property in order to use ES modules, but the idea is that NodeJS would be executed with a _default module format_, which could be itself implemented via a flag for example `node --esModules x.js`, while defaulting to CommonJS for backwards-compatibility. Over time it may be possible to ultimately make this flag the default.

### Loading some modules in the package as CommonJS

For a package that contains both ES modules in a `lib` folder and CommonJS modules in a `test` folder, the approach that could be taken would be to have two package.json files - one at the base of the package, and another in the `test` folder itself, explicitly specifying `"format": "cjs"`. The `test` folder package.json would then take precedence for that subfolder, allowing a gradual adoption path.

### Deprecation path for explicit format

To make such a process possible, npm (which already modifies package.json files on install) could automatically include a `"format": "cjs"` in all installed packages that don't explicitly specify this property. In this way, legacy packages could eventually all be relied on to be installed with a `"format": "cjs"` present in their package.json within the node_modules install folder, and the default module flag mode of NodeJS could eventually be switched (over LTS release deprecation path timescales). Of course such a path would be reliant on npm assistance.
