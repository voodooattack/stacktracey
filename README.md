# StackTracey

[![Build Status](https://travis-ci.org/xpl/stacktracey.svg?branch=master)](https://travis-ci.org/xpl/stacktracey) [![Windows Build Status](https://ci.appveyor.com/api/projects/status/cvuygb8grrvm1sdm?svg=true)](https://ci.appveyor.com/project/xpl/stacktracey) [![Coverage Status](https://coveralls.io/repos/github/xpl/stacktracey/badge.svg?branch=master)](https://coveralls.io/github/xpl/stacktracey) [![NPM](https://img.shields.io/npm/v/stacktracey.svg)](http://npmjs.com/package/stacktracey) [![Scrutinizer Code Quality](https://img.shields.io/scrutinizer/g/xpl/stacktracey.svg)](https://scrutinizer-ci.com/g/xpl/stacktracey/?branch=master) [![dependencies Status](https://david-dm.org/xpl/stacktracey/status.svg)](https://david-dm.org/xpl/stacktracey)

Parses call stacks. Reads sources. Clean & filtered output. Sourcemaps. Node & browsers.

## Why

- [x] Simple
- [x] Works in Node and browsers, \*nix and Windows
- [x] Allows hiding library calls / ad-hoc exclusion (via `// @hide` marker)
- [x] Provides source text for call locations
- [x] Fetches sources synchronously (even in browsers) via [get-source](https://github.com/xpl/get-source)
- [x] Full sourcemap support
- [x] Extracts useful information from `SyntaxError` instances
- [x] [Pretty printing](https://github.com/xpl/stacktracey/#pretty-printing)
      <img width="898" alt="screen shot 2017-09-27 at 16 53 46" src="https://user-images.githubusercontent.com/1707/30917345-79899004-a3a4-11e7-8d48-e217e2d5e2cd.png">

## What For

- [Better exception reporting](https://github.com/xpl/ololog#pretty-printing-error-instances) for Node and browsers
- [Advanced logging](https://github.com/xpl/ololog#displaying-call-location) (displaying call locations)
- Assertion printing

## How To

```bash
npm install stacktracey
```

```javascript
StackTracey = require ('stacktracey')
```

Captures the current call stack:

```javascript
stack = new StackTracey ()            // captures the current call stack
```

Parses stacks from an `Error` object:

```javascript
stack = new StackTracey (error)       // parses error.stack
stack = new StackTracey (error.stack) // raw string
```

It is an array instance:

```javascript
stack instanceof Array // returns true
stack.length           // num entries
stack[0]               // top
```

...where each item exposes:

```javascript
{
    beforeParse:  <original text>,
    callee:       <function name>,
    calleeShort:  <shortened function name>,
    file:         <full path to file>,       // e.g. /Users/john/my_project/node_modules/foobar/main.js
    fileRelative: <relative path to file>,   // e.g. node_modules/foobar/main.js
    fileShort:    <short path to file>,      // e.g. foobar/main.js
    fileName:     <file name>,               // e.g. main.js
    line:         <line number>,             // starts from 1
    column:       <column number>,           // starts from 1

    index:          /* true if occured in HTML file at index page    */,
    native:         /* true if occured in native browser code        */,
    thirdParty:     /* true if occured in library code               */,
    hide:           /* true if marked as hidden by "// @hide" tag    */,
    syntaxError:    /* true if generated from a SyntaxError instance */
}
```

Accessing sources:

```javascript
stack = stack.withSources // will return a copy of stack with all items supplied with sources
top   = stack[0]          // top item
```

...or:

```javascript
top = stack.withSource (0) // supplies source for an individiual item
```

...or:

```javascript
top = StackTracey.withSource (stack[0]) // supplies source for an individiual item
```

It will return an item supplied with the source code info (already mapped through sourcemaps):

```javascript
{
    ... // all the previously described fields

    line:       <original line number>,
    column:     <original column number>,
    sourceFile: <original source file object>,
    sourceLine: <original source line text>
}
```

To learn about the `sourceFile` object, read the [get-source](https://github.com/xpl/get-source#get-source) docs.

## Cleaning Output

```javascript
stack = stack.withSources.clean
```

1. Excludes locations marked with the `isThirdParty` flag (library calls)
2. Excludes locations marked with a `// @hide` comment (user defined exclusion)
3. Merges repeated lines (via the `.mergeRepeatedLines`)

You can augment the global `isThirdParty` predicate with new rules:

```javascript
StackTracey.isThirdParty.include (path => path.includes ('my-lib')) // paths including 'my-lib' will be marked as thirdParty
```
```javascript
StackTracey.isThirdParty.except (path => path.includes ('jquery')) // jquery paths won't be marked as thirdParty
```

P.S. It is better to call `.clean` on stacks supplied with sources (i.e. after the `.withSources`), to make the `// @hide` magic work, and to make third-party recognition work by reading proper file names in case if your source is compiled from other sources (and has a sourcemap attached).

## Pretty Printing

```javascript
const prettyPrintedString = new StackTracey (error).pretty
```

It produces a nice compact table layout (thanks to [`as-table`](https://github.com/xpl/as-table)), supplied with source lines (if available):

```
at shouldBeVisibleInStackTrace     test.js:25                 const shouldBeVisibleInStackTrace = () => new StackTracey ()
at it                              test.js:100                const stack = shouldBeVisibleInStackTrace ()                
at callFn                          mocha/lib/runnable.js:326  var result = fn.call(ctx);                                  
at run                             mocha/lib/runnable.js:319  callFn(this.fn);                                            
at runTest                         mocha/lib/runner.js:422    test.run(fn);                                               
at                                 mocha/lib/runner.js:528    self.runTest(function(err) {                                
at next                            mocha/lib/runner.js:342    return fn();                                                
at                                 mocha/lib/runner.js:352    next(suites.pop());                                         
at next                            mocha/lib/runner.js:284    return fn();                                                
at <anonymous>                     mocha/lib/runner.js:320    next(0);                  
```

You can even replace the default NodeJS exception printer with this! This is how you can do it:

```javascript
process.on ('uncaughtException',  e => { /* print the stack here */ })
process.on ('unhandledRejection', e => { /* print the stack here */ })
```

## Parsing `SyntaxError` instances

For example, when trying to `require` a file named `test_files/syntax_error.js`:

```javascript
// next line contains a syntax error (not a valid JavaScript)
foo->bar ()
```

...the pretty printed call stack for the error thrown would be something like:

```
at (syntax error)                  test_files/syntax_error.js:2  foo->bar ()
at it                              test.js:184                   try { require ('./test_files/syntax_error.js') }
at runCallback                     timers.js:781
at tryOnImmediate                  timers.js:743
at processImmediate [as _immediat  timers.js:714
```

...where the first line is generated from parsing the raw output from the `util.inspect` call in Node. Unfortunately, this won't work in older versions of Node (v4 and below) as these versions can't provide any meaningful information for a `SyntaxError` instance.

## Array Methods

All StackTracey instances expose `map`, `filter`, `concat`, `reverse` and `slice` methods. These methods will return mapped, filtered, joined, reversed and sliced stacks, respectively:

```javascript
s = new StackTracey ().slice (1).filter (x => !x.isThirdParty) // current stack shifted by 1 and cleaned from library calls

s instanceof StackTracey // true
s instanceof Array       // true
```

Other methods of the `Array` are supported too, but they will return `Array` instances, not StackTracey instances. You can convert from array via this:

```javascript
stack = new StackTracey (array)
```

..and to array via this (but generally this is not needed — you can pass around StackTracey instances as if they were real Arrays):

```javascript
Array.from (stack)
```

## Extra Stuff

You can compare two locations via this predicate (tests `file`, `line` and `column` for equality):

```javascript
StackTracey.locationsEqual (a, b)
```

Resetting source cache (calls `getSource.resetCache ()` from [get-source](https://github.com/xpl/get-source)):

```javascript
StackTracey.resetCache ()
```

## Projects That Use StackTracey

- [Ololog](https://github.com/xpl/ololog) — a better `console.log` for the log-driven debugging junkies!
- [CCXT](https://github.com/ccxt-dev/ccxt) — a cryptocurrency trading library that supports 85+ exchanges
- [pnpm](https://github.com/pnpm/pnpm) — a fast, disk space efficient package manager (faster than npm and Yarn!)

