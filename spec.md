This specification represents a NodeJS resolution algorithm which is backwards compatible with the existing [Node modules resolution algorithm](https://nodejs.org/api/modules.html#modules_all_together), while supporting package.json lookups for all module loads, allowing the parent module type and `"module"` property to assist in determining the distinction between ES Modules and CommonJS.

For performance the package.json contents are cached for the duration of execution (including caching the absence of a package.json file), just like modules get cached in the module registry for the duration of execution. This caching is described in the GET_PACKAGE_CONFIG function.

The `Module` object here returned by the resolver would effectively be a wrapper class of around the [V8 Module class](https://v8.paulfryzel.com/docs/master/classv8_1_1_module.html).

Note also that this resolver would only apply within the ES module system, while `require()` would continue to use the legacy resolver code. This way we do not disturb the edge cases of this algorithm on the existing ecosystem. This resolver thus handles only the logic branch path of assuming a package with both a `module` and a `main` property should provide its modules as ES Modules, as CommonJS modules would be assumed for `require()` calls into the package by the legacy resolver algorithm.

> **RESOLVE(name: String, parentPath: String): Module**
> 1. Assert _parentPath_ is a valid file system path.
> 1. If _name_ is a NodeJS core module then,
>    1. Return the NodeJS core _Module_ object.
> 1. If _name_ is a valid absolute file system path, or begins with _'./'_, _'/'_ or '../' then,
>    1. Let _requestPath_ be the path resolution of _name_ to _parentPath_, with URL percent-decoding applied and any _"\\"_ characters converted into _"/"_ for posix environments.
>    1. Return the result of _RESOLVE_MODULE_PATH(requestPath)_, propagating any error on abrupt completion.
> 1. Otherwise, if _name_ parses as a _URL_ then,
>    1. If _name_ is not a valid file system URL then,
>       1. Throw _Invalid Module Name_.
>    1. Let _requestPath_ be the file system path corresponding to the file URL.
>    1. Return the result of _RESOLVE_MODULE_PATH(requestPath)_, propagating any error on abrupt completion.
> 1. Otherwise,
>    1. Return the result of _NODE_MODULES_RESOLVE(name)_, propagating any error on abrupt completion.

> **RESOLVE_MODULE_PATH(requestPath: String): Module**
> 1. Let _{ main, module, packagePath }_ be the destructured object values of the result of _GET_PACKAGE_CONFIG(requestPath)_, propagating any errors on abrupt completion.
> 1. Let _loadAsModule_ be equal to _false_.
> 1. If _module_ is equal to _true_ then,
>    1. Set _main_ to _undefined_.
>    1. Set _loadAsModule_ to _true_.
> 1. If _module_ is a string then,
>    1. Set _main_ to _module_.
>    1. Set _loadAsModule_ to _true_.
> 1. If _main_ is not _undefined_ and _packagePath_ is not _undefined_ and is equal to the path of _requestPath_ (ignoring trailing path separators) then,
>    1. Set _requestPath_ to the path resolution of _main_ to _packagePath_.
> 1. Let _resolvedPath_ be the result of _RESOLVE_FILE(requestPath)_, propagating any error on abrubt completion.
> 1. If _resolvedPath_ is not _undefined_ then,
>    1. If _resolvedPath_ ends with _".mjs"_ then,
>       1. Return the resolved module at _resolvedPath_, loaded as an ECMAScript module.
>    1. If _resolvedPath_ ends with _".json"_ then,
>       1. Return the resolved module at _resolvedPath_, loaded as a JSON file.
>    1. If _resolvedPath_ ends with _".node"_ then,
>       1. Return the resolved module at _resolvedPath_, loaded as a NodeJS binary.
>    1. If _resolvedPath_ ends with _".wasm"_ then,
>       1. Throw _Invalid Module Name_.
>    1. If _loadAsModule_ is set to _true_ then,
>       1. Return the resolved module at _resolvedPath_, loaded as an ECMAScript module.
>    1. If the module at _resolvedPath_ contains a _"use module"_ directive then,
>       1. Return the resolved module at _resolvedPath_, loaded as an ECMAScript module.
>    1. Otherwise,
>       1. Return the resolved module at _resolvedPath_, loaded as a CommonJS module.
> 1. Throw _Not Found_.

> **GET_PACKAGE_CONFIG(requestPath: String): { main: String, format: String, packagePath: String }**
> 1. For each parent folder _packagePath_ of _requestPath_ in descending order of length,
>    1. If there is already a cached package config result for _packagePath_ then,
>       1. If that cached package result is an empty configuration entry then,
>          1. Continue the loop.
>       1. Otherwise,
>          1. Return the cached package config result for this folder.
>    1. If _packagePath_ ends with the segment _"node_modules"_ then,
>       1. Break the loop.
>    1. If _packagePath_ contains a _package.json_ file then,
>       1. Let _json_ be the parsed JSON of the contents of the file at "${packagePath}/package.json", throwing an error for _Invalid JSON_.
>       1. Let _main_ be the value of _json.main_.
>       1. If _main_ is defined and not a string, throw _Invalid Config_.
>       1. Let _module_ be the value of _json.module_.
>       1. If _module_ is defined and not a string or boolean, throw _Invalid Config_.
>       1. Let _result_ be the object with keys for the values of _{ main, module, packagePath }_.
>       1. Set in the package config cache the value for _packagePath_ as _result_.
>       1. Return _result_.
>    1. Otherwise,
>       1. Set in the package config cache the value for _packagePath_ as an empty configuration entry.
> 1. Return the empty configuration object _{ main: undefined, module: undefined, packagePath: undefined }_.

> **RESOLVE_FILE(filePath: String): String**
> 1. If _filePath_ is a file, return _X_.
> 1. If _"${filePath}.mjs"_ is a file, return _"${filePath}.mjs"_.
> 1. If _"${filePath}.js"_ is a file, return _"${filePath}.js"_.
> 1. If _"${filePath}.json"_ is a file, return _"${filePath}.json"_.
> 1. If _"${filePath}.node"_ is a file, return _"${filePath}.node"_.
> 1. If _"${filePath}/index.js"_ is a file, return _"${filePath}/index.js"_.
> 1. If _"${filePath}/index.json"_ is a file, return _"${filePath}/index.json"_.
> 1. If _"${filePath}/index.node"_ is a file, return _"${filePath}/index.node"_.
> 1. Return _undefined_.

> **NODE_MODULES_RESOLVE(name: String, parentPath: String): String**
> 1. For each parent folder _modulesPath_ of _parentPath_ in descending order of length,
>    1. Let _resolvedModule_ be the result of _RESOLVE_MODULE_PATH("${modulesPath}/node_modules/${name}/")_, propagating any errors on abrupt completion.
>    1. If _resolvedModule_ is not _undefined_ then,
>       1. Return _resolvedModule_.
