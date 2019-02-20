# `--type=auto` Module Type Detection Algorithm

This is the algorithm, in psuedocode, for the `--type=auto`/`-a` Node startup option described [here](./README.md). It is based on detailed syntax rules outlined [here](https://github.com/nodejs/node-eps/issues/57#issuecomment-300870976).

This algorithm relies on the fact that it is evaluating an initial entry point—the very first file loaded for a program. Affirmatively detecting CommonJS would be virtually impossible otherwise, as it would no longer be a safe assumption that a reference to a globally defined `require` confirms that a file is CommonJS (since a file loaded earlier in the program could have defined `require` in the global scope). Additionally, many files (either CommonJS or ESM) that are not entry points are often ambiguous, for example containing only `console.log` statements or references to user-defined globals. For this algorithm to be both confident (no false positives) and inclusive (few files judged ambiguous) it must be restricted to initial entry points.

## Step 1: Parse

The initial entry point file must be parsed so that its syntax may be analyzed.

## Step 2: Look for signifiers of ESM JavaScript

If the source code contains either of the following:

- An `import` statement, like `import "x";`

- An `export` statement, like `export {};`

Then the file is unambiguously ESM. Stop and execute it as ESM JavaScript.

These are the only two syntactical elements that are unambiguously ESM. The `import` expression, `import("x")`, is allowed in CommonJS.

If the source code did not contain an ESM signifier, continue:

## Step 3: Look for signifiers of CommonJS JavaScript

If the source code contains any of the following:

- A reference to a variable named `__filename` without a declaration for that variable.

- A reference to a variable named `__dirname` without a declaration for that variable.

- A reference to a variable named `require` without a declaration for that variable.

- A reference to a variable named `module` without a declaration for that variable.

- A reference to a variable named `exports` without a declaration for that variable.

- A reference to `arguments` in the top scope.

- A reference to `return` in the top scope.

Then the file is unambiguously CommonJS. Stop and execute it as CommonJS JavaScript.

There are additional syntactical elements that are unambiguously part of the _script goal;_ but while all CommonJS files are script goal JavaScript, not all script goal JavaScript files are CommonJS. (For example, script goal JavaScript intended for browsers.) While we could detect syntax that is unambiguously script goal, as opposed to ESM’s module goal, we exclude those signifiers because they don’t necessarily signify CommonJS. Script goal signifiers are also very uncommon, such as the `with` keyword, and are unlikely to appear in user code.

If the source code did not contain a CommonJS signifier, continue:

## Step 4: Throw

Source code that lacks unambiguous signifiers of ESM syntax or CommonJS references cannot be reliably determined to be either module type, and therefore the algorithm throws an error:

```
Error: The module type of file.js cannot be unambiguously determined.
```

Alternatively, we may decide to evaluate ambiguous files as either ESM or as CommonJS, but for now the safest approach is to exit.