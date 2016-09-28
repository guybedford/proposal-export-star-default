# ECMAScript Proposal: Include default export in `export * from 'module'`

**Stage:** 0

**Author:** Guy Bedford

**Reviewers:** Caridy Patiño

**Specification:** http://guybedford.github.io/proposal-export-star-default/

## Motivation

`export * from 'module'` statements provide a way to package together many modules into
a top-level entry-point.

In the current specification, `export * from 'module'` will export all named exports, **except for
the default export**.

There are use cases though where it is useful to be able to have the `default` export defined within
an internal private module, and have that exposed through `export *`.

The biggest concern with including `default` in `export *` is how to handle collisions, but `export *`
is [already designed to handle ambiguity well](#ambiguity).

## Proposal

The proposal is to remove special-casing of `default` for `export *` in the current specification
in order to ensure `export *` more closely resembles a spread conceptually and simplify the symmetry of the export syntaxes.

See http://guybedford.github.io/proposal-export-star-default/ for the diff.

## Ambiguity

### Collisions

When two `export *` statements in a module resolve to the same export name, a SyntaxError is thrown during [instantiation](http://www.ecma-international.org/ecma-262/6.0/#sec-moduledeclarationinstantiation).

This same behaviour would then apply to default exports through `export *`:

a.js
```javascript
export default 'a';
export let someIdentifier = 'x';
```
b.js
```javascript
export default 'b';
export let anotherIdentifier = 'y';
```

c.js
```javascript
export * from './a.js';
export * from './b.js'; // throws a SyntaxError
```

The SyntaxError in the above ensures that collisions do not happen, avoiding ambiguity.

### Resolving Collisions

In order to resolve the previous ambiguity, we can use the same process we do for
other named exports, which is to rely on the fact that local exports always
override `export *` ambiguity:

```javascript
export * from './a.js';
export * from './b.js';

// ensure that the default is specifically exported from "a"
export default from './a.js';
```

where `a.js` and `b.js` are exactly as in the previous example.

> The above syntax is based on the [export default from proposal](https://github.com/leebyron/ecmascript-export-default-from),
although any method of setting the local default export directly would work here.

The inclusion of the explicit default export then avoids the previous ambigous SyntaxError from being thrown.

In this way, we can fully resolve and control export star resolution ambiguities, with natural equivalence between the named exports
and default exports.
