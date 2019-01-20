# Built-in decorators

This proposal suggests that, as we reach for syntax for a way to modify some other syntactic production, JavaScript could use a variant of the decorator syntax, of the form `@scope: expressionLike`. This syntactic space would be analogous to Allen Wirfs-Brock's proposal to use `keyword.pseudoproperty`, in the vein of `new.target` and `function.sent`.

## The problem

Why add new syntax? All else being equal, it'd be nice to stop adding more syntax to JavaScript. There's already a lot to learn in the language, and syntax has the particular property that it's in your face: it can't just do the right thing in the background, since you have to actually type it out to invoke it. It's not just based on composing existing constructs--it's a new construct.

However, we just keep running into the need to make modified versions of other constructs. A unified syntax for such extensions is no excuse for adding unnecessary complexity, but it could make it more straightforward to design and prototype solutions that *are* justified, while minimizing the complexity of the JavaScript grammar long-term.

### We keep wanting to add new forms that modify old forms

Some recently specified forms, which you could think of as a modification of another form, include:
- async functions, generators, and async generators
- getters, setters, and static class elements
- let and const, as variants on var
- export declarations
- tagged templates

Some future possibilities for the need for syntactic forms which are modified versions of other constructs
- Proposals discussed in TC39
  - [Class, method and field decorators](https://github.com/tc39/proposal-decorators/)
  - [Function.prototype.toString censorship](https://github.com/domenic/proposal-function-prototype-tostring-censorship)
  - Defensible classes
  - [Syntactic tail calls](https://github.com/tc39/proposal-ptc-syntax/)
  - [Extensible numeric literals](https://github.com/tc39/proposal-extended-numeric-literals)
  - ["use module"](https://github.com/tc39/proposal-modules-pragma)
- Other proposals which add new syntactic constructs, which might be considered modified forms at a stretch
  - [do expressions](https://github.com/tc39/proposal-do-expressions)
  - [protocols](https://github.com/michaelficarra/proposal-first-class-protocols)
  - [pattern matching](https://github.com/tc39/proposal-pattern-matching)
  - [explicit resource management](https://github.com/tc39/proposal-using-statement)
  - [asset references](https://github.com/sebmarkbage/ecmascript-asset-references)
  - [class static block](https://github.com/rbuckton/proposal-class-static-block#readme)
- More far-out ideas not discussed in committee
  - [Operator overloading `with operators from` declarations](https://github.com/littledan/proposal-operator-overloading/)
  - Value type or typed object literals and declarations
  - Package public/private declarations

We likely won't standardize all of these, but we may want to move forward with some of them. It seems likely that even more such proposals will come in the future.

### Existing solutions aren't working anymore

Some new syntax features can be designed by using existing keywords. However, there are very few remaining reserved JavaScript keywords to ascribe meaning to--`enum` may be the only one that's not used at all.

#### Contextual keywords

In the past, we've dealt with that issue using "contextual keywords"--tokens which are keywords in one context, while remaining a variable name in other contexts. However, we're running out of space and energy to keep going with contextual keywords.

Recently added new contextual keywords like `let`, `async` and `await` created a combination of additional lookahead (with corresponding difficulty in maintaining parser efficiency) and surprising edge cases where they are treated as a variable instead. Even in cases that work out relatively cleanly, like `yield`, it's somewhat weird to have to look at the immediately enclosing function to tell whether it's a keyword or a variable (and there are still some non-trivial edge cases, e.g., with nested arrow functions). And the edge cases often result in differences between JavaScript implementations that take time to iron out.

The committee's goal of maintaining a non-backtracking grammar significantly constrains what kinds of contextual keywords are possible. Cover grammars to deal with the situation are getting more and more complex. It just doesn't seem to be sustainable to keep going this way.

#### String-based pragmas

JavaScript's existing well-known mode switch pragma is `"use strict";`. Proposals for similar string-based pragmas:
- ["use module"](https://github.com/tc39/proposal-modules-pragma)
- [Function.prototype.toString censorship](https://github.com/domenic/proposal-function-prototype-tostring-censorship): `"use no Function.prototype.toString";`

Such declarations might work for these particular cases, but they runs into the following issues:
- Typos can't be caught through the runtime semantics, only by tools; pragmas with a typo are simply ignored. By contrast, type errors or variable typos might not be caught ahead of time, but if you have tests with good code coverage, you are likely to observe some kind of error; tests for these declarations would have to be written to test against the specific effect fo that decorator.
- As these pragmas are inside of a string, they can't really use any of JavaScript's syntactic constructs, such as taking a parameter which is a variable reference.
- Composability is awkward: At least the grammar of `"use strict"` only interprets it as the first statement in a block; it's unclear how/whether multiple pragmas would be parsed (though this may be solvable).
- They are in an awkward position, *within* a block as opposed to preceding it. This leads to the requirement for certain restrictions, such as functions with non-simple parameters not being allowed to contain a `"use strict"` pragma.

#### Overloading existing keywords and constructs

Although most keywords and punctuation are taken, or planned to be taken, there are many combinations of existing tokens which are currently not permitted in JavaScript's grammar, and therefore can likely be used in new ways. For example, ES6 reuses the `for` keyword in the new `for (const variable of iterable) { }` construct. Several proposals repurpose existing keywords in new grammatical constructs:
- [The `return continue` STC proposal](https://github.com/tc39/proposal-ptc-syntax/)
- [Resource usage based on `try`](https://github.com/tc39/proposal-using-statement/issues/20)
- [Pattern matching](https://github.com/tc39/proposal-pattern-matching)
- [`do` expressions](https://github.com/tc39/proposal-do-expressions)
- [Operator overloading `with operators from` declarations](https://github.com/littledan/proposal-operator-overloading/)

It might be notable that it's hard to find historical examples of this technique, but it's all over the place in proposals! Repurposing existing keywords for something which is semantically fairly different risks being confusing for programmers. The JavaScript learner could ask, "but what does the `try` keyword actually do?" when the answer is "it's repurposes for multiple things which are only loosely related". That might not be such a good experience, even if it leads to something which is syntactically unambiguous and easy to parse without weird edge cases.

## The idea of this proposal

Let's see if we can reduce the syntax variation of new proposals by introducing a new pattern which can be flexibly used in many of these different potential future language features: to modify an existing construct with a new feature, prepend it with a built-in decorator `@scope: decorator`

### Credits for the idea

Thanks to Ron Buckton for suggesting this particular syntax in [this decorators issue comment](https://github.com/tc39/proposal-decorators/issues/69#issuecomment-385609889).

Thanks to Leo Balter for raising the idea of using a variant of decorator syntax for syntactic tail calls, which was really helpful for getting me thinking on this path.

Thanks to Yehuda Katz for hia insight in bringing decorators to JavaScript and vision to use them as a basis for modifying all soets of syntactic constructs, and for discussions and reviews of this proposal.

Thanks to Waldemar Horwat, Kevin Gibbons, Mark Miller, Leo Balter, Dave Herman and more in TC39 for raising the urgency and priority of limiting the growth of syntactic complexity in JavaScript.

### Intuition for built-in decorators

From the perspective of a programmer who uses decorators, and doesn't dive into implementing their own, decorators are a construct for modifying the behavior of the decorated code. Built-in decorators, including special built-in ones which have a `scope:` prefix, are doing just that. The `@` is a hint that this modification is going on, and what follows it describes what's being modified.

### Related usage in other languages

Decorator-like constructs do different things in different languages; it's far from unusual to have built-in decorators which have magic powers, even alongside a rich language with various metaprogramming constructs.
- Swift [Attributes](https://docs.swift.org/swift-book/ReferenceManual/Attributes.html) provide a fixed set of modifiers--it is purely a namespace for built-in constructs using familar `@attribute`-style syntax.
- Rust [Attributes](https://doc.rust-lang.org/reference/attributes.html) provide a built-in set of modifiers, plus a `#[derive(XYZ)]` attribute for a developer-provided `XYZ`, where you can define your own custom `derive` behavior for a trait.
- C# [Attributes](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/concepts/attributes/) include (optional) "targets" before a `:`, similarly to this proposal. Several built-in attributes are provided which control fundamental things such as the loading, security, and JIT behaviors attached to code.

### Benefits of a unified syntax for modification

The goal here is not to encourage an explosion of new modifiers that everyone has to learn. But, when new modifiers do turn out to be needed in the evolution of the language, JavaScript developers should not have to learn a new set of syntactic rules to govern it, which are specific to that particular modifier.

## Examples of application

This section contains some extremely early ideas (not necessarily reviewed by the proposal champions!) for how some TC39 proposals could make use of the built-in decorator syntax proposed in this document.

As a common scope for static, built-in decorators, this article contains many examples with `@use:` because:
- Many JS developers are familiar with the `"use strict";` directive, and `@use:` builds on that intuition.
- It's a short word which is easy to type.
- It seems to sound reasonable when reading the code out loud.

For constructs which are a mode for constructs which define functions, classes, methods, etc (including expressions for those constructs), `@def:` is used because:
- Many developers are familiar with the `def` abbreviation for `define` from Python
- This prefix evokes more the idea of choosing a specific different definition form, whereas `use` sounds more like a mode tweak.

(Happy to bikeshed about future other names in an issue.)

### Function.prototype.toString censorship

<table><tr><th><a href="https://github.com/domenic/proposal-function-prototype-tostring-censorship">Current proposal</a> form<th>As a built-in decorator
<tr><td>

```js
"use no Function.prototype.toString";
function f() { }
function g() { }
```
<td>

```js
@use: noFunctionPrototypeToString;
function f() { }
function g() { }
```
<tr><td>

```js
function h() {
  "use no Function.prototype.toString";
}
```
<td>

```js
@use: noFunctionPrototypeToString
function h() { }
```
</table>

There is a particular question about how "deep" should these built-in decorator syntaxes go--should such a construct be able to apply recursively, into nested constructs, e.g., inner functions? Although user-level programmatic decorators may never get that ability, on further reflection, I don't see why built-in built-in decorator syntax should not be able to have those semantics. See [previous discussion](https://github.com/domenic/proposal-function-prototype-tostring-censorship/issues/13).

Note that the addition of this syntax would not meet the "backwards compatibility" goal of the Function.prototype.toString proposal--it would be a syntax error to use it in browsers that do not support it; see [this issue](https://github.com/domenic/proposal-function-prototype-tostring-censorship/issues/9) for discussion.

### Operator overloading declarations

<table><tr><th><a href="https://github.com/littledan/proposal-operator-overloading/">Current proposal</a> form<th>As a built-in decorator
<tr><td>

```js
with operators from Scalar, Vector;
function f() { return scalar + vector; }
```
<td>

```js
@use: operators(Scalar, Vector);
function f() { return scalar + vector; }
```
</table>

### Syntactic tail calls

ES2015's implicit proper tail calls (PTC, also referred to as "tail call optimization") has only been shipping in JavaScriptCore (Safari) and not other JavaScript engines used in web browsers because of an unresolved disagreement about whether it's OK to have tail call semantics be opted into by default.

The ["Syntactic Tail Calls"](https://github.com/tc39/proposal-ptc-syntax/) (STC) proposal adds explicit syntax for tail calls. However, this proposal uses the syntax `return continue`, which not everyone is happy about. Built-in decorators could be used for an alternative syntax design.

<table><tr><th>Variant<th>Syntax<th>As a built-in decorator
  <tr><td><a href="https://github.com/tc39/proposal-ptc-syntax/">Callsite-based</a><td>

```js
function factorial(n, acc = 1) {
  if (n === 1) {
    return acc;
  }

  return continue factorial(n - 1, acc * n)
}
```
<td>

```js
function factorial(n, acc = 1) {
  if (n === 1) {
    return acc;
  }

  @use: tailcall
  return factorial(n - 1, acc * n)
}
```
<tr><td><a href="https://github.com/tc39/proposal-ptc-syntax/issues/5">Function-based</a><td>

```js
function factorial(n, acc = 1) {
  "use tail calls";
  if (n === 1) {
    return acc;
  }

  return factorial(n - 1, acc * n)
}
```
<td>

```js
@use: tailcall
function factorial(n, acc = 1) {
  if (n === 1) {
    return acc;
  }

  return factorial(n - 1, acc * n)
}
```
</table>

In both cases, the `@use: tailcall` built-in decorator would be an assertion that the marked code contains a tail call, and the semantics the affected callsites would be treated as such. The placement is determined by the programmer, depending on their programming style and intuition. Decorating the return statement is a more specific assertion and marks the specific callsite, whereas decorating the whole function can be convenient when there may be many tail call sites, and it would be repetitive to mark each one.

### Variants on classes

Different kinds of syntaxes have been tossed around for "defensible classes", "typed objects", "value types", etc. These other kinds of classes logically relate to classes somehow (they have methods, fields, may use private, etc), but have somewhat different semantics. Because these constructs are intended to be reliable, it makes sense to have a solid, built-in way to trigger them, and one way is with dedicated syntax.

Rather than introduce various kinds of contextual keywords, we could use built-in decorators for this sort of case:

<table><tr><th>Proposal<th>Syntax<th>As a built-in decorator
  <tr><td><a href="https://github.com/hemanth/es-next#defensible-classes">Defensible classes</a><td>

```js
const class Point { 
  constructor(x, y) {
    public getX() { return x; }
    public getY() { return y; }
  }
  toString() { 
    return `<${this.getX()}, ${this.getY()}>`;
  }
}
```
<td>

```js
@def: defensible
class Point { 
  #x; #y;
  constructor(x, y) {
    this.#x = x;
    this.#y = y;
  }
  getX() { return this.#x; }
  getY() { return this.#y; }
  toString() { 
    return `<${this.getX()}, ${this.getY()}>`;
  }
}
```
<tr><td><a href="https://github.com/tschneidereit/proposal-typed-objects/blob/master/explainer.md">Typed objects<td>

```js
const Point = new StructType([
  { name: "x", type: float64 },
  { name: "y", type: float64 }
]);
```
<td>

```js
@def: typed
class Point {
  @typed: float64
  x;
  @typed: float64
  y;
}
```
<tr><td>Value types<td>

```js
value class Point { 
  #x; #y;
  constructor(x, y) {
    this.#x = x;
    this.#y = y;
  }
  getX() { return this.#x; }
  getY() { return this.#y; }
  toString() { 
    return `<${this.getX()}, ${this.getY()}>`;
  }
}
```

<td>

```js
@def: value
class Point { 
  #x; #y;
  constructor(x, y) {
    this.#x = x;
    this.#y = y;
  }
  getX() { return this.#x; }
  getY() { return this.#y; }
  toString() { 
    return `<${this.getX()}, ${this.getY()}>`;
  }
}
```
</table>

#### Auto-bound methods

Many of the use-cases for decorators have to do with various ways to "auto-bind" methods, that is, make it so that `this.method` refers to `this.method.bind(this)`. Class fields permit a pattern for using arrow functions in initializers, but this causes runtime overhead by pre-allocating the bound method per instance, when a bound method may not be needed. The decorator-based patterns can make that allocation lazy, but still, a bound method needs to be created each time a method is used, even if it's called immediately.

As a built-in feature, the runtime may be able to eliminate the creation of the bound method in the cases where it's called immediately, creating and caching it only when needed. It also may be more efficient to run during startup if it's built-in to the IC system rather than calling out to a custom getter (though in optimized code, the distinction may not exist).

<table><tr><th>Usage style<th>As a built-in decorator
<tr><td>On a specific method<td>

```js
class C {
  @use: autobind
  method() { }
}
```
<tr><td>On all methods in a class<td>

```js
@use: autobind
class C {
  method() { }
}
```
</table>

### Explicit resource management

The [explicit resource management](https://github.com/tc39/proposal-using-statement) proposal creates a protocol and convenient syntax for disposing of resources after they are used in a block of code. 

<table><tr><th>Usage mode<th>Syntax<th>As a built-in decorator
<tr><td>Using an allocated resource in a block<td>

```js
let file = File.open(path);
using (file) {
  file.doStuff();
}
```
<td>

```js
let file = File.open(path);
@use: resource(file) {
  file.doStuff();
}
```
<tr><td>Binding for the resource live in a block<td>

```js
using (let file = File.open(path)) {
  file.doStuff();
}
```
<td>

```js
{
  @use: resource
  let file = File.open(path);
  file.doStuff();
}
```
<tr><td>Stand-alone statement<td>
---
<td>

```js
{
  let file = File.open(path);
  @use: resource(file);
  file.doStuff();
}
```
</table>

In this case, the built-in decorator `@use: resource` can either decorate a block, a variable declaration, or an empty statement, but serves an intuitively similar purpose in all cases.

### Module import variants

The [asset references](https://github.com/sebmarkbage/ecmascript-asset-references) introduces a new import-like construct, with the `asset` contextual keyword. This keyword faces similar problems to other contextual keywords, with the need for cover grammars, ASI hazards, and a thing that looks like `import` but is subtlely different syntactically. 

Similarly, there have been various ideas about providing CORS or SRI-related metadata inline with module imports. Further thoughts about hashing have led to the refinement that signatures or hashes are better kept out-of-line (to avoid having to rehash/resign up the whole tree because a dependency changes), but there could be other kinds of metadata needed to pass into an import.

<table><tr><th>Proposal<th>Syntax<th>As a built-in decorator
<tr><td>Asset references<td>

```js
asset Foo from "./foo.mjs"; 
async function load() {
   // The type system now knows that this is wrong:
  let foo: {bar: string} = await import(Foo);
}
```
<td>

```js
@asset: reference
import Foo from "./foo.mjs"; 
async function load() {
   // The type system now knows that this is wrong:
  let foo: {bar: string} = await import(Foo);
}
```
<tr><td>Import metadata<td>

```js
import Foo from "./foo.mjs" with option "value", option2 "value2";
```
<td>

```js
@use: options({ option: "value", option2: "value2" })
import Foo from "./foo.mjs";

```
</table>

### Custom code transforms

Many people are experimenting with using various kinds of code generation in JavaScript, e.g., with [Babel](https://babeljs.io/) plugins. Babel's parser only permits standards-track syntax (plus JSX and type systems) to be parsed, so transforms end up giving new semantics to existing syntax. Sometimes, the semantics provided can be considered in line with standard JavaScript, and sometimes, they may be somewhat distinct. Because there's no syntactic place to "opt in" to other semantics, new interpretations of existing constructs are the only option.

There's a composability issue here: If these transforms are done globally, rather than specifically to just constructs that need them, the code base and the transform sort of go together, and it might not be possible to develop two different types of code together, if the transforms need to be used on the file as a whole.

To resolve this, the built-in decorator `@scope: expression` construct could be used to specifically call for a transform on a particular piece of code which is affected.  These would all be syntax errors executed in JavaScript without transformation by tools (since only known scopes with specification-defined behavior are accepted), without including their own custom syntax. 

#### Managing scope names

There is some risk that different non-standard scope names may overlap and conflict, much like different libraries that add properties of the same name to the global object may overlap and conflict. Unlike in libraries, there's no clear way to "import" these constructs to disambiguate (though one could imagine an analogous, tools-only "import type"-style construct for built-in decorators); built-in decorators aren't based on lexical scoping. This would be unfortunate, but at least it would be a limited-impact conflict (to those particular constructs).

Some mitigation strategies to reduce the risks of overlaps:
- By convention, scopes which are project-specific could use related names, such as `@ember:` or `@ts:`, could be used to call out to tool- or project-specific transforms, allowing these plugins to be used together in the same code bases.
- We could build a central place to share information about what kinds of scope names are used by projects; when choosing a new scope name, people can consult this list and choose one not included, and then note their choice on the list. Participation would be optional but beneficial to participants. This work could take place in the [JS shared interfaces repository](https://github.com/littledan/js-shared-interfaces/).

## Syntactic details

The grammar of a built-in decorator is based on ordinary decorators:

```
  DecoratorList[Yield, Await]:
    DecoratorList[?Yield, ?Await]opt Decorator[?Yield, ?Await]
    DecoratorList[?Yield, ?Await]opt BuiltinDecorator[?Yield, ?Await]

  Decorator[Yield, Await]:
    @ DecoratorMemberExpression[?Yield, ?Await]
    @ DecoratorCallExpression[?Yield, ?Await]

  BuiltinDecorator[Yield, Await]:
    @ IdentifierName : DecoratorMemberExpression[?Yield, ?Await]
    @ IdentifierName : DecoratorCallExpression[?Yield, ?Await]

  DecoratorMemberExpression[Yield, Await]:
    IdentifierReference[?Yield, ?Await]
    DecoratorMemberExpression[?Yield, ?Await] . IdentifierName
    ( Expression[+In, ?Yield, ?Await] )

  DecoratorCallExpression[Yield, Await]:
    DecoratorMemberExpression Arguments[?Yield, ?Await]
```

`DecoratorList`s can be potentially used in various contexts, preceding the syntactic productions `Statement` (omitting `ExpressionStatement`, see below), `ArrayLiteral`, `ObjectLiteral`, `FunctionExpression`, `ClassExpression`, `GeneratorExpression`, `AsyncFunctionExpression`, `AsyncGeneratorExpression`, `ArrowFunction`, and `AsyncArrowFunction`.

Initially, "Static Semantics: Early Errors" rules are used to maintain a syntax error in all cases of using built-in decorators, and all cases where decorators exist that aren't supported by the [class decorators](https://github.com/tc39/proposal-decorators/) proposal. Future proposals may chip away at this space, defining various subsets, while leaving the rest in tact for further proposals. The encouraged path is to never give an interpretation to the entire `@IdentifierName:` space, and instead chose individual `IdentifierName`s to define, to leave space open for later

#### ASI hazard for `EmptyStatement`

One use case for built-in decorators is a statement-like pragma, which covers the rest of the program, as in `@use: operators(Vector);` above. In this case, the semicolon is mandatory, otherwise the immediately following construct will be decorated. I believe that it should be OK to require a semicolon in a fixed construct like this.

An alternative is to say that the `IdentifierName` determines whehter it's an `EmptyStatement` decorator or one which takes a larger syntactic construct afterwards. This option is dispreferred, as it weakens the usability of built-in decorators as an extension point for tools (those tools would have to pass in this parameter to the parser).

#### Cover grammar for decorated arrow functions

Arrow function parameters will require a certain cover grammar, to differentiate from a function call. I think it will be similar in shape to the grammar of async arrow functions, and it will hopefully be able to reuse some grammar components, but more work is needed to on the details.

#### Omitting other literals

In theory, we could apply this to strings (`@foo "bar"`) or numeric literals (`@foo 3`). However, these are already handled by other language features or proposals:
- Rather than decorated string literals, we have [tagged template literals](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Template_literals), which provide more power.
- Rather than decorated numeric literals, we have [the extended numeric literals proposal](https://github.com/tc39/proposal-extended-numeric-literals), which has more terse syntax (needed for tiny numbers).

### No support for decorating arbitrary expressions

It probably wouldn't make sense to apply built-in decorators to arbitrary expressions (`@scope: foo baz(bing)`). If parentheses are used around the expression, there's no clear way to make this syntactically unambiguous with `foo(baz(bing))` as a call, and the intention is that no particular knowledge of the scope is needed to parse what comes after it.

## Alternatives considered

Aside from the current strategies cited earlier, there has been some discussion of other syntaxes for certain modified constructs:
- Instead of the `@use: decorator` namespace, we could use the more abbreviated syntax `@@decorator`.
- For value object literals, the syntax `#{ }` could be used, rather than something like `@use: value { }`.
- Keywords following `@` are disallowed, so constructs like `@const` for defensible classes is a possible area for expansion, though more limited.
- Decorators for certain constructs which don't currently permit decorators  could be interpreted as built-in by default, without a scope prefix (though this cuts off a potentially important evolutionary path where we later do have semantics for procedural decorators).

There is no contradiction between continuing to define certain specialized syntaxes and setting aside a built-in syntax fallback. We can initially prototype features using scoped built-in decorator syntax, and later consider these more terse syntaxes for features where we decide it makes more sense.

## Next steps

This proposal doesn't need to be at any particular stage, but if it's a direction that looks generally attractive, we might consider the following:
- Ensure that the placement of decorators doesn't close off any particular future paths
- When proposing another syntactic construct for JavaScript, consider whether it could use built-in decorator syntax.
- Adding parser support for built-in decorators (and general metaproperty syntax) into tools like Babel, so that transform plugins can experiment with applying semantics to it.

### Implications for the class decorators proposal

- Class decorators should probably come after `export`, as we may want a built-in decorator to modify `export` declarations themselves, even if it seems implausible to have a procedural decorator modifying `export`.
- There are many use cases for built-in decorators modifying deeply nested constructs, so we should make sure to build a clear intuition for this among decorators users. In the context of such an intuition, it makes sense for ordinary class decorators to be able to access private class elements.
