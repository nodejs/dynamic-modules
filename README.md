# ECMAScript Proposal: Dynamic Modules

## Background

The semantics of ECMAScript 262 Modules are based on [Abstract Module Records](https://tc39.github.io/ecma262/#sec-abstract-module-records), but currently only a single implementation of these are provided through [Source Text Module Records](https://tc39.github.io/ecma262/#sec-source-text-module-records).

Various types of dynamic module record have been proposed in the past, from the dynamic module records originally supporting zebra striping relations with CommonJS to the [strawman module mutator proposal](https://gist.github.com/dherman/fbf3077a2781df74b6d8),
which was designed to be supported alongside a module registry API.

This specification defines a new Dynamic Module Record which can be programatically created and defined by host environments to support tree-order execution of non-source-text module records. In addition these Dynamic Module Records support
late definition of named export bindings at execution time to support named exports for legacy module format execution.

## Motivation

Host environments such as Node.js need to define module records that are not based on source text but created programatically.

Examples for Node.js include builtin modules (while these are still CommonJS), native addons, JSON files, and CommonJS. Currently NodeJS generates a source text internally to handle these cases, which is an unwieldy approach.

In addition implementing named exports support for CommonJS in Node.js is [currently blocked](https://github.com/nodejs/node/pull/16675) due to the limitations of the source text wrapper approach requiring the exported names to be known before execution.

## Proposed Solution

The approach taken is to define a Dynamic Module Record which implements all the expected fields and methods of an Abstract Module Record.

The DynamicModuleRecord calls out to a host hook, `HostEvaluateDynamicModule` on execution.

In addition, module execution is allowed to define named export bindings during execution using a provided `SetDynamicExportBinding` concrete method.

Because the Dynamic Module Record cannot have any dependencies itself, early accesses to bindings before execution are not possible. If any bindings remain uninitialized after execution, a `ReferenceError` is thrown, effectively moving this validation phase from instantiate to post-execution.

> Note: **Dynamic Module Records have no dependencies.** This provides support for a single one-way boundary without further transitive dependencies on other modules in the graph, avoiding circular reference behaviours. This is distinct from integration with WASM or Binary AST which can have further transitive dependencies with the existing graph, therefore needing their own Abstract Module Record implementations.

## ECMAScript 262 Changes

The specification here relies on some minor changes in ECMA-262 itself, which are currently tracking in PR https://github.com/tc39/ecma262/pull/1306.

## Illustrative Example

Consider the Source Text Module Record "main.js":

```js
import { readFile } from 'fs';
import * as fs from 'fs';
readFile('./x.js');
fs.readFile('./y.js');
```

where `'fs'` is a CommonJS module being implemented as a Dynamic Module Record.

The following rough outline steps are taken in the spec:

2. `'fs'` is instantiated, which for Dynamic Module Records is mostly a noop apart from book-keeping.
1. `'main.js'` is instantiated.
2. The `readFile` binding is initialized, at this point a new lexical binding is created for it in the lexical environment for `'fs'`.
4. The `fs` Namespace Exotic Object is created, with a single export name, `readFile`.
5. `'fs'` is evaluated, calling the `HostEvaluateDynamicModule` hook. The host can then execute the CommonJS module using the third-party loader, and set the lexical environment values for named export bindings using the `SetDynamicExportBinding` concrete method.
  It may, for example, also define a `writeFile` binding at this point.
6. If any bindings for `'fs'` are uninitialized at this point, a `ReferenceError` is thrown. Eg if there was a `import { missing } from 'fs'`.
6. If the `fs` Namespace Exotic Object is defined (it is created lazily in the existing spec), any new export names are added to it at this point. There is no worry of early access here as it is impossible for user code to read the object before this point.
7. `'main.js'` is evaluated, and accesses the `readFile` binding as well as the `fs.readFile` namespace binding. All bindings are guaranteed to be defined at this point.

While the exact implementation is not specified, the host API defining `'fs'` might look something like:

```js
function executeDynamicModule (id, setDynamicExportBinding) {
  const exports = require(id);
  for (const exportName of Object.keys(exports)) {
    setDynamicExportBinding(exportName, exports[exportName]);
  }
  // in addition `setDynamicExportBinding` can be stored and used for runtime mutations
}
```

## Handling Namespace and Star Exports

Some spec changes need to be made to support namespace star exports from Dynamic Modules.

Consider the following example:

lib.js
```js
export * from 'dynamic-module';
```

main1.js
```js
import { dynamicMethod } from './lib.js';
```

main2.js
```js
import * as lib from './lib.js';
```

`'main1.js'` can be supported fine, as the `ResolveExport` concrete method will ensure a placeholder for `dynamicMethod` that is then validated on execution.

On the other hand, the namespace object for `'main2.js'` will not know the list of exported names from `'dynamic-module'` when it is created during instantiation.

In order to support this, we introduce some book-keeping to track any Namespace Exotic Objects created that reference star exports of Dynamic Module Records.

The namespace is then initially created without any exports, and as soon as all dynamic modules star exported by the namespace are finished executing, then
the namespace is finalized with its export names being set.

### Uninstantiated circular edge case

The only case where it is possible to observe unexecuted dynamic modules is in special cases of cycles like the following:

a.mjs
```js
import './b.mjs';
export * from 'dynamic';
console.log('a exec');
```

b.mjs
```js
import * as a from './a.mjs';
console.log('b exec');
console.log(Reflect.ownKeys(a));
```

In the example above, importing a.mjs will result in b.mjs being executed before both a.mjs and dynamic. As a result, the namespace object will have no exports during this intermediate phase and can be thought
of as a form of TDZ on the module namespace exports.

The important thing is that partially populated exports are never visible so that we don't get runtime-dependent export visibility behaviours.
Previously throwing behaviour was implemented for this case, but the error wouldn't be very easy to debug. Also there are many valid cases where the observability would never be seen.

## FAQ

### Why not support constant bindings?

These bindings effectively behave like let bindings. Constant bindings would provide utility if there were a means to mutate them, but that currently isn't possible in this spec approach.

### Why not support hoisted function bindings?

There is no need to provide hoisted bindings for dynamic modules since they will always execute before their consumer.

### Does this mean that the host will define named exports based on what a user requests?

While the export bindings are created lazily based on the importer, any export bindings not explicitly initialized immediately throw after execution.

In addition the `HostEvaluateDynamicModule` hook explicitly is required not to depend on any importer cues of what bindings have been imported.

By ensuring `HostEvaluateDynamicModule` populates exports independently, and also throwing a `ReferenceError` after execution for undefined bindings, we continue to guarantee
the well-defined nature of export names despite supporting late definition here.

### Why does this need to be a host method?

A Reflect-style interface could be considered for exposing programmatic module creation, and this proposal could form a base for that.

But the goal was to create a minimal proposal solving the immediate needs rather than a whole new JS API that would have its own design concerns.

Such a feature could certainly build on top of this to go with eg any registry APIs in future.

### Could dynamic module records not be defined outside of ECMA-262?

To support dynamic modules that execute in the tree-order with export names possibly defined at execution time, it is not possible to use a source text
wrapper approach at all - a new engine API is required to support this.

This specification for Dynamic Module Records takes a number of steps that are not clear they would be supported by the ECMA-262 module semantics:

* We are allowing the `ResolveExport` concrete method to define let-style export binding placeholders when called on dynamic modules to ensure availability during instantiate.
* We are moving the validation of export names for Dynamic Modules from the instantiate phase to the post-execution phase.
* We are possibly extending new export names onto Namespace Exotic Objects after they have already been created (from a spec perspective).
* To ensure dynamic modules have executed before their importers, we need to throw a reference error within the ES instantiation algorithm in certain specific circular reference edge cases.
* In order to handle book-keeping on which Namespace Exotic Objects need this extension, we add a new parameter to `GetExportNames` tracking the requesting module.

In addition, this work may provide the foundations for exposing a Reflect-base dynamic module API in future, as previously considered.

## Specification

* [Ecmarkup source](https://github.com/nodejs/dynamic-modules/blob/master/spec.html)
* [HTML version](https://nodejs.github.io/dynamic-modules/)

## Implementations

An initial implementation in v8 has been developed at https://github.com/v8/v8/compare/master...guybedford:dynamic-modules,
and is undergoing further review and feedback. Collaboration welcome.
