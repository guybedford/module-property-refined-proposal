This specification represents a NodeJS resolution algorithm which is backwards compatible with the existing [Node modules resolution algorithm](https://nodejs.org/api/modules.html#modules_all_together), while supporting package.json lookups for all module loads, allowing a new `format` property to indicate `"cjs"` or `"esm"` formats for packages to be interpreted as CommonJS or ECMAScript Modules respectively.

For performance, it is assumed that the algorithm would cache the package.json contents in a single cache,
reusing the cached contents for the duration of execution (including caching the absence of a package.json file), just like modules get cached in the module registry for the duration of execution.

> **RESOLVE(name: String, parentPath: String): ModuleNamespace**
> 1. Assert _parentPath_ is a valid file system path.
> 1. If _name_ is a NodeJS core module then,
>    1. Return the NodeJS core _ModuleNamespace_ object.
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

> **RESOLVE_MODULE_PATH(requestPath: String): ModuleNamespace**
> 1. Let _{ main, format, packagePath }_ be the destructured object values of the result of _GET_PACKAGE_CONFIG(requestPath)_, propagating any errors on abrupt completion.
> 1. If _format_ is _undefined_ then,
>    1. Set _format_ to the current execution environment default module format name.
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
>    1. If _format_ is equal to _"cjs"_ then,
>       1. Return the resolved module at _resolvedPath_, loaded as a CommonJS module.
>    1. If _format_ is equal to _"esm"_ then,
>       1. Return the resolved module at _resolvedPath_, loaded as an ECMAScript module.
> 1. Throw _Not Found_.

> **GET_PACKAGE_CONFIG(requestPath: String): { main: String, format: String, packagePath: String }**
> 1. For each parent folder _packagePath_ of _requestPath_ in descending order of length,
>    1. If _packagePath_ ends with the segment _"node_modules"_ then,
>       1. Break the loop.
>    1. If _packagePath_ contains a _package.json_ file then,
>       1. Let _json_ be the parsed JSON of the contents of the file at "${packagePath}/package.json", throwing an error for _Invalid JSON_.
>       1. Let _main_ be the value of _json.main_.
>       1. Let _format_ be the value of _json.format_.
>       1. If _main_ or _format_ is defined and not a string, throw _Invalid Config_.
>       1. If _format_ is defined and not equal to _"cjs"_ or _"esm"_ then, throw _Invalid Config_.
>       1. Return the object with keys for the values of _{ main, format, packagePath }_.
> 1. Return the empty configuration _{ main: undefined, format: undefined, packagePath: undefined }_.

> **RESOLVE_FILE(filePath: String): String**
> 1. If _filePath_ is a file, return _X_.
> 1. If _"${filePath}.mjs"_ is a file, return _"${filePath}.js"_.
> 1. If _"${filePath}.js"_ is a file, return _"${filePath}.js"_.
> 1. If _"${filePath}.json"_ is a file, return _"${filePath}.json"_.
> 1. If _"${filePath}.node"_ is a file, return _"${filePath}.node"_.
> 1. If _"${filePath}/index.js"_ is a file, return _"${filePath}/index.js"_.
> 1. If _"${filePath}/index.json"_ is a file, return _"${filePath}/index.json"_.
> 1. If _"${filePath}/index.node"_ is a file, return _"${filePath}/index.node"_.
> 1. Return _undefined_.

> **NODE_MODULES_RESOLVE(name: String, parentPath: String): String**
> 1. For each parent folder _modulesPath_ of _parentPath_ in descending order of length,
>    1. Let _resolvedModule_ be the result of _RESOLVE_PATH("${modulesPath}/node_modules/${name}")_, propagating any errors on abrupt completion.
>    1. If _resolvedModule_ is not _undefined_ then,
>       1. Return _resolvedModule_.
