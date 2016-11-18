# ECMAScript Proposal: Include default export in `export * from 'module'`

**Stage:** 0

**Author:** Guy Bedford

**Reviewers:** Caridy Patiño

**Specification:** http://guybedford.github.io/proposal-export-star-default/

## Motivation

In the current specification, `export * from 'module'` will export all named exports, **except for
the default export**.

There are use cases though where it is useful to be able to have the `default` export defined within
an internal private module, and have that exposed through `export *`.

### 1. Symmetry with named exports

  Consider a module X with both a named export and a default export that is re-exported through another module.
  If we explicitly list its export names when re-exporting them in `a.js`, we can import everything as expected.
  But if we use export * in `b.js`, the default export is not accessible, breaking the user intuition that `default`
  is a named export like any other.

  x.js
  ```javascript
  export default 'default';
  export let name = 'name';
  ```

  a.js
  ```javascript
  export { name, default } from './x.js';
  ```

  b.js
  ```javascript
  export * from './x.js';
  ```

  ```javascript
  import { name } from './a.js';
  import { default } from './a.js';

  import * as A from './a.js';
  A.default;
  // -> 'default'
  ```

  And yet if we try the same with `b.js`:
  ```javascript
  import { name } from './b.js';
  import { default } from './b.js';
  // Syntax Error: 'default' is not exported

  import * as B from './b.js';
  B.default
  // -> undefined
  ```


### 2. Using a top-level exporting `index` module

  A convention in NodeJS is to have an `index.js` which is the main entry point of the package.

  If we want to create this module using `export *` statements to expose modules from sub-folders, we
  cannot export the default in this way:

  index.js
  ```javascript
  export * from './lib/package-core.js';
  export let extraInfo = 'abc';
  ```

  If `lib/package-core.js` contained a default export, we would not be exposing it.

  We would need to explitly export the default with something like:

  index.js
  ```javascript
  export * from './lib/package-core.js';
  export { default } from './lib/package-core.js';
  export let extraInfo = 'abc';
  ```

  The issue here is that if we were ever to remove the default export from
  `lib/package-core.js`, we would then get a SyntaxError in `index.js` that the
  default export does not exist anymore. The change of removing the default export
  would need to be made in two separate places.

  The above pattern is also already widely seen in `index.js` modules on npm, of the form:

  ```javascript
  module.exports = require('./lib/package.js');
  ```

### 3. Dynamic wrapping

  The case for blindly exporting the defaut can also be extended to use cases where we want to dynamically
  generate a module wrapper exposing the same exports as another module, without knowing in advance its named exports.

  Consider a use case such as an npm CDN, which allows shortcut URLs like `https://npmcdn.com/lodash`,
  which then expose a module at a full versioned URL like `https://npmcdn.com/lodash@4.17.2/index.js`.

  A dynamically generated module for `https://npmcdn.com/lodash` could look like:

  ```javascript
  export * from './lodash@4.17.2/index.js';
  ```

  but if there was a default export we would have to provide a different response:

  ```javascript
  export * from './lodash@4.17.2/index.js';
  export { default } from './lodash@4.17.2/index.js';
  ```

  If we don't include the default export when it is needed, we may miss the `default`, and if do we include
  it when it isn't needed, we will get an error.

  This restriction has thus added a new static analysis concern to what would otherwise be a simple server response.

## Proposal

The proposal is to remove special-casing of `default` for `export *` in the current specification
in order to simplify the symmetry of `export *` as well as to enable easier module wrapping.

The change is backwards-compatible with the existing specification, disabling the syntax error for importing
a default export through an `export *`, and making `default` available as a named export on the module namespace object for this case.

http://guybedford.github.io/proposal-export-star-default/

## Example: Handling Ambiguity

One concern with including `default` in `export *` is how to handle collisions, but `export *`
is already designed to handle ambiguity well - when two `export *` statements in a module resolve the
same export name, a SyntaxError is thrown as in [15.2.1.16.4 12.d.ii)](https://tc39.github.io/ecma262/#sec-moduledeclarationinstantiation):

  x.js
  ```javascript
  export default 'x';
  export let name = 'nameX';
  ```

  y.js
  ```javascript
  export default 'y';
  export let name = 'nameY';
  ```

  z.js
  ```javascript
  export * from './x.js';
  export * from './y.ys';
  ```

When we try to import `name` via

  ```javascript
  import { name } from './z.js';
  ```

we will get a SyntaxError that name is ambiguous.

This conflict can be resolved though by explicitly selecting a module to export from

  z.js
  ```javascript
  export * from './x.js';
  export * from './y.ys';
  export { name } from './x.js';
  ```

Now when we import `name`, it is correctly resolved.

With this spec change, the same ambiguity resolution process would apply to the default export:

  ```javascript
  import { default } from './z.js';
  ```

would throw a SyntaxError.

  z.js
  ```javascript
  export * from './x.js';
  export * from './y.ys';
  export { default } from './x.js';
  ```

would then resolve that SyntaxError by providing an explicit precedence, providing symmetry between handling conflicts between named exports
and default exports via `export *`.
