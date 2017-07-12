# NodeJS Module Resolver Specification Proposal - Refined "module" Property Handling

This proposal specifies the `"module"` property in the package.json, building on the previous work in the [In Defense of Dot JS](https://github.com/dherman/defense-of-dot-js/blob/master/proposal.md) proposal, as well as many other discussions.

Instead of supporting the additional `"modules"` and `"modules.root"` properties from that proposal, this proposal aims to adjust the handling of `"module"` slightly so that it is the only property supported.

A draft specification of the NodeJS module resolution algorithm with this adjustment is included at [spec.md](spec.md).

### Motivation

Currently all our build tools detect modules in slightly different ways. The `package.json` `module` property has gained some traction as an entry point, but there isn't currently clarity on how exactly this property behaves in the edge cases. Since tools are currently driving the ecosystem conventions, it is worth refining the exact conventions with an active specification that can gain support, so that we can continue to converge on the module contract in NodeJS, and do our best to avoid incompatibilities in future.

### Module property cases

Instead of trying to consider a unified resolver here, we break the behaviour of NodeJS resolution into two separate resolvers:
* The current resolver as in use today, which will continue to be used to resolve CommonJS modules from CommonJS modules.
* The new ES Modules resolver, that also has the ability to load CommonJS modules.

When using CommonJS `require`, the legacy resolver would be applied, and when using ES modules, the new ES module resolver algorithm, as along the lines specified here would be applied.

_The basic rule is then simply that the ES module resolver always loads a module from package with a "module" property as an ES module, and loads a module from a package without that property as a CommonJS module (unless it is a .mjs file or "use module" source)._

Under this rule, the simple cases remain the same as the In Defense of Dot JS Proposal:

* A package with only a `main` and no `module` property will be loaded as containing CommonJS modules only.
* A package with only a `module` property and no `main` property will be loaded as containing ES Modules only.

The difficult case with the In Defense of JS Proposal is the transition case of a package that contains both a `main` and `module` property - selecting which main entry point and target to use when loading `pkg` or `pkg/x.js`.

For a package that contains both a `main` and a `module` property -
* When the parent module doing the require is an ES Module, the `module` main will apply, and any module loaded from the package will be loaded as an ES Module.
* When the parent module doing the require is a CommonJS module, the `main` main will apply, and any module loaded from the package will be loaded as a CommonJS Module.

In this way, we continue to support the existing ecosystem with backwards compatibility, while keeping the scope of the specification as simple as possible.

### Public API for mixed CJS and ES Module packages

A package delivering both CommonJS and ES Modules would then typically tell its users to just import via `import {x} from 'pkgName'` or `require('pkgName').x`.

In the case where a package publicly exposes sub-modules, it would need to document that the CommonJS and ES Module sources are at different paths - `import {x} from 'pkgName/submodule.js'` vs `import {x} from 'pkgName/cjs/submodule.js'`. Or simply a `.js` and `.mjs` variant, this being the author's preference.

### Package boundary detection

* For a given module, the package.json file is checked in that folder, continuing to check parent folders for a package.json if none is found. If we reach a parent folder of `node_modules`, we stop this search process.
* When no package.json module property is found, NodeJS would default to loading any module as CommonJS. This would throw for attempting to load an ES module file with no package.json module property present (no [unambiguous grammar implementation](https://github.com/bmeck/UnambiguousJavaScriptGrammar/blob/master/README.md) being provided here).
* When loading an `.mjs` file or a source with a `"use module"`, the format is implied by the extension, loading as an ECMAScript module.

These rules are taken into account in the [draft resolver specification here](spec.md).

### Loading modules without a package.json

If writing a `.js` file without any `package.json` configuration, it would be possible to opt-in to ES modules by indicating this by either using the `.mjs` file extension or `"use module"` directive (https://github.com/tc39/proposal-modules-pragma).

### Packages consisting of both CommonJS and ES Modules

For a package that contains both ES modules in a `lib` folder and CommonJS modules in a `test` folder, the approach that could be taken would be to have two package.json files - one at the base of the package with a `package.json` containing a `module` property, and another in the `test` folder itself, without any `module` property. The `test` folder package.json would then take precedence for that subfolder, allowing a partial adoption path.

### Enabling wasm

For future support of Web Assembly, this spec also reserves the file extension `.wasm` as throwing an error when attempting to load modules with this extension, in order to allow Web Assembly loading to work by default in future.
