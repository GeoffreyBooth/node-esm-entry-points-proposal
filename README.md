# Support for ESM syntax in entry points in Node.js

**Contributors**: Geoffrey Booth (@GeoffreyBooth), Guy Bedford (@guybedford), John-David Dalton (@jdalton), Jan Krems (@jkrems), Saleh Abdel Motaal (@SMotaal), Bradley Meck (@bmeck)

## Overview

This proposal aims to define how Node should determine the module format (CommonJS or ESM) for the following entry point types:

- Direct file with extension, e.g. `node file.js`

- extensionless files, e.g. `/usr/local/bin/npm`

- `--eval` and `--print`, e.g. `node --eval 'console.log("hello")'`

- `STDIN`, e.g. `echo 'console.log("hello")' | node`

This proposal covers only the module type of the _entry point._ Once the entry point is loaded, if it is ESM then the determination of parse goals of imported files is covered by the [Import File Specifier Resolution proposal](https://github.com/GeoffreyBooth/node-import-file-specifier-resolution-proposal/). CommonJS entry points use Node’s existing behavior.

### Package scope

The [Import File Specifier Resolution proposal](https://github.com/GeoffreyBooth/node-import-file-specifier-resolution-proposal/) introduces the concept of a _package scope._ A package scope is a folder containing a `package.json` file and all of that folder’s subfolders except those containing other `package.json` files and those folders’ subfolders. See [example](https://github.com/GeoffreyBooth/node-import-file-specifier-resolution-proposal/#example).

## Proposal

The following methods can tell Node how to interpret the initial entry point:

1. Entry point files with `.mjs` extensions are parsed as ESM.

1. Entry point files with `.cjs` extensions are parsed as CommonJS.

1. Entry point files with `.js` extensions or are extensionless are parsed as ESM if they are within an ESM [package scope](#package-scope), or CommonJS otherwise.

1. Entry point files that are symlinks are parsed as ESM or CommonJS depending on whether the _target_ is in an ESM package scope or has an explicit file extension.

1. If Node is run with the `--type=module` or `-m` command line flag, the entry point is parsed as ESM.

1. If Node is run with the `--type=commonjs` command line flag, the entry point is parsed as CommonJS.

1. If Node is run with the `--type=auto` or `-a` command line flag, Node detects the module format of the entry point and evaluates it accordingly.

### `.mjs` file extension

Files with `.mjs` extensions are always parsed as ESM, regardless of package scope. If a flag is used that conflicts with the extension, like `node --type=commonjs file.mjs`, an error is thrown.

### `.cjs` file extension

Files with `.cjs` extensions are always parsed as CommonJS, regardless of package scope. If a flag is used that conflicts with the extension, like `node --type=module file.cjs`, an error is thrown.

The `.cjs` extension is needed because otherwise there would be no way to create explicitly CommonJS files inside an ESM package scope. In a CommonJS package, one can deep import an `.mjs` file to load it as ESM despite the package’s CommonJS scope, e.g. `import 'cjs-package/src/file.mjs'`. The `.cjs` extension allows the inverse, e.g. `import 'esm-package/dist/file.cjs'`.

### `.js` and extensionless files as initial entry point

Per the [package scope algorithm](https://github.com/GeoffreyBooth/node-import-file-specifier-resolution-proposal/#procedure), if the entry point is a `.js` file (e.g. `node file.js`) the path to `file.js` is searched for the closest `package.json` and that `package.json` is read to see if it has an [ESM-signifying field](https://github.com/GeoffreyBooth/node-import-file-specifier-resolution-proposal/#parsing-packagejson). If it does, `file.js` is parsed as ESM.

For example, say you have a folder `~/Sites/cool-app`, and in it the files `package.json` and `app.js`. If `package.json` has a field that defines its package scope to be ESM, `node app.js` will run as ESM; otherwise it will run as CommonJS.

The above also applies to extensionless files. `node_modules/typescript/bin/tsc`, for example, would evaluate as ESM if `node_modules/typescript/package.json` contains an ESM-signifying field.

### Symlinks

A common case is a globally installed package like NPM, which creates a symlink like `/usr/local/bin/npm`. In NPM’s case, on macOS at least that path is a symlink to `/usr/local/lib/node_modules/npm/bin/npm-cli.js`.

To handle extensionless files like `npm` that are symlinks, the search for package scope will begin at the symlink’s _target._ So in NPM’s case, Node will search for a `package.json` in `/usr/local/lib/node_modules/npm/bin/`, then `/usr/local/lib/node_modules/npm/` (where it finds one).

Alternatively, if the symlink’s target was a file with an `.mjs` extension, it would be evaluated as ESM directly without the package scope search. (Likewise for `.cjs` extensions being evaluated as CommonJS.)

For now there is simply no way to use ESM in an extensionless file that is not a symlink and is not in an ESM package scope, aside from invoking it via `node` with a flag (e.g. `node -m /usr/local/bin/npm`) or setting `--type=module` to the `NODE_OPTIONS` environment variable.

### `--type=module` flag

The command line flags `--type=module` and `-m` tell Node to parse as ESM entry points that would otherwise be ambiguous (`.js` and extensionless files, string input via `--eval` or `STDIN`). For example:

```bash
node --type=module --eval 'import { sep } from "path"; console.log(sep)'

echo 'import { sep } from "path"; console.log(sep)' | node -m

NODE_OPTIONS='--type=module' node --eval 'import { sep } from "path"; console.log(sep)'

export NODE_OPTIONS='--type=module';
node --eval 'import { sep } from "path"; console.log(sep)'
```

The name `--type=module` was chosen to match the Web’s `<script type="module">`.

### `--type=commonjs` flag

The command line flag `--type=commonjs` tell Node to parse as CommonJS entry points that would otherwise be ambiguous (`.js` and extensionless files, string input via `--eval` or `STDIN`). For example:

```bash
node --type=commonjs --eval 'const { sep } = require("path"); console.log(sep)'
```

### `--type=auto` flag

The command line flags `--type=auto` and `-a` tell Node to detect the module format for potentially ambiguous entry points (`.js` and extensionless files, string input via `--eval` or `STDIN`). The algorithm for this is as follows:

1. Parse the source code of the initial entry point.

2. If the source code is unambiguously ESM (has `import` or `export` statements, etc.) evaluate as ESM. *Else:*

3. Evaluate as CommonJS.
