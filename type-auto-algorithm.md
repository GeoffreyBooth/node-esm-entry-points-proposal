# `--type=auto` Module Type Detection Algorithm

This is the algorithm, in psuedocode, for the `--type=auto`/`-a` Node startup option described [here](./README.md). It is based on detailed syntax rules outlined [here](https://github.com/nodejs/node-eps/issues/57#issuecomment-300870976).

This algorithm relies on the fact that it is evaluating an initial entry point—the very first file loaded for a program. Affirmatively detecting CommonJS would be virtually impossible otherwise, as it would no longer be a safe assumption that a reference to a globally defined `require` implies that a file is CommonJS (since a file loaded earlier in the program could have defined `require` in the global scope). Additionally, many files (either CommonJS or ESM) that are not entry points are often ambiguous, for example containing only `console.log` statements or references to user-defined globals. For this algorithm to be both confident (no false positives for ESM, no realistic false positives for CommonJS) and inclusive (few files judged ambiguous) it must be restricted to initial entry points.

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

- A reference to a global variable named `__filename`, without code defining the variable.

- A reference to a global variable named `__dirname`, without code defining the variable.

- A reference to a global variable named `require`, without code defining the variable.

- A reference to a global variable named `module`, without code defining the variable.

- A reference to a global variable named `exports`, without code defining the variable.

Then the file is assumed to be CommonJS. Stop and execute it as CommonJS JavaScript.

There are syntactical elements that are unambiguously part of the _script goal;_ but while all CommonJS files are script goal JavaScript, not all script goal JavaScript files are CommonJS. (For example, script goal JavaScript intended for browsers.) While we could detect syntax that is unambiguously script goal, as opposed to ESM’s module goal, we exclude those signifiers because they don’t necessarily signify CommonJS. Script goal signifiers are also very uncommon, such as the `with` keyword, and are unlikely to appear in user code.

Technically, ESM JavaScript initial entry point files can include references to global variables and still be valid ESM JavaScript. An file containing `require('fs');` is valid ESM, for example, though of course it would throw upon evaluation. A file containing `if (require) { /* ... */ }` would evaluate successfully. Because of this, it is impossible to unambiguously detect CommonJS with total certainty; however, the authors of this proposal cannot identify any real-world use cases for an ESM JavaScript initial entry point to reference CommonJS globals. (There is the use case of JavaScript written for transpilation that contains both `import` and `require` statements, but that’s not valid ESM JavaScript.) Unless a significant class of realistic code can be identified that is both valid ESM and uses meaningful references to CommonJS globals, the heuristic of looking for references to CommonJS globals should affirmatively detect all practical CommonJS files. Especially as an opt-in feature, we consider the utility of `--type=auto` to outweigh concerns regarding detecting CommonJS with total certainty.

If the source code did not contain a CommonJS signifier, continue:

## Step 4: Throw

Source code that lacks the above signifiers of ESM syntax or CommonJS references cannot be reliably determined to be either module type, and therefore the algorithm throws an error:

```
Error: The module type of file.js cannot be reliably determined.
```

Alternatively, we may decide to evaluate ambiguous files as either ESM or as CommonJS, but for now the safest approach is to exit.
