# NodeJS Module Resolver Proposal - package.json "format" property

This proposal specifies a `"format"` property in the package.json to set the default module format for interpreting the modules within that package.

This proposal builds on previous work such as [In Defense of Dot JS](https://github.com/dherman/defense-of-dot-js/blob/master/proposal.md) as well as other discussions on package.json [properties](https://github.com/nodejs/node/wiki/ES6-Module-Detection-in-Node#option-4-meta-in-packagejson) and [flags](https://github.com/dherman/defense-of-dot-js/issues/10).

The `"format"` property takes two possible values - `"esm"` for ECMAScript Modules, and `"cjs"` for CommonJS modules.

When loading any module, the package.json is checked using a small extension to the existing NodeJS resolution algorithm, and the resultant format is applied as the default when loading any modules from that package. This package.json format property does not apply through `node_modules` folder boundaries.

A draft specification of the NodeJS module resolution algorithm with this adjustment is included at [spec.md](spec.md).

### Format interpretation

* The assumption is that NodeJS has a default of interpreting modules as CommonJS when no other indication is given.
* For a given module, the package.json file is checked in that folder, continuing to check parent folders for a package.json if none is found. If we reach a parent folder of `node_modules`, we stop this search process.
* When no package.json format is found, NodeJS would default to loading a module as CommonJS. This would throw for attempting to load an ES module file with no package.json format present (no [unambiguous grammar implementation](https://github.com/bmeck/UnambiguousJavaScriptGrammar/blob/master/README.md) being provided here).
* When loading an `.mjs` file, the format is implied by the extension, loading as an ECMAScript module.

These rules are taken into account in the [draft resolver specification here](spec.md).

### Loading modules without a package.json

When starting a new NodeJS project, the package.json would need a ``"format": "esm"`` property to load `.js` files as modules instead of as CommonJS.

There are two possible ways of working around this:

1. Write a file with an `.mjs` file extension.
2. Write a `.js` file with a `"use module"` directive (https://github.com/tc39/proposal-modules-pragma).

### Packages consisting of both CommonJS and ES Modules

For a package that contains both ES modules in a `lib` folder and CommonJS modules in a `test` folder, the approach that could be taken would be to have two package.json files - one at the base of the package, and another in the `test` folder itself, explicitly specifying `"format": "cjs"`. The `test` folder package.json would then take precedence for that subfolder, allowing a partial adoption path.

### Enabling future extensions

For future support of Web Assembly, this spec also reserves the file extension `.wasm` as throwing an error when attempting to load modules with this extension, in order to allow Web Assembly loading to work by default in future.

By using a string value for the module contract, additional adjustments to the module contract might be supported in future to alter things like entry point resolution, automatic file extension adding and file extension handling.
