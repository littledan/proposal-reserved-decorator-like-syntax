# Reserved decorator-like syntax

This proposal suggests that, as we reach for syntax for a way to modify some other syntactic production, JavaScript could use **decorator-like syntax**: a variant of the decorator syntax, of the form `@[expressionLike]`, and we could reserve this space in advance for future use.

Status: Not at a stage; not yet introduced to TC39. On the agenda for the January 2019 meeting.

## The problem

*Why add new syntax?* All else being equal, it'd be nice to stop adding more syntax to JavaScript. There's already a lot to learn in the language, and syntax has the particular property that it's in your face: it can't just do the right thing in the background, since you have to actually type it out to invoke it. It's not just based on composing existing constructs--it's a new construct.

However, we just keep running into the need to make modified versions of other constructs. A unified syntax for such extensions is no excuse for adding unnecessary complexity, but it could make it more straightforward to design and prototype solutions that *are* justified, while minimizing the complexity of the JavaScript grammar long-term.

### We keep wanting to add new forms that modify old forms

Some recently specified forms, which you could think of as a modification of another form, include:
- async functions, generators, and async generators
- getters, setters, and static class elements
- let and const, as variants on var
- export declarations
- tagged templates

Many TC39 proposals also modify other existing forms; see the applications section below for some examples of how decorator-like syntax could be used for them. We likely won't standardize all of these proposals, but we may want to move forward with some of them. It seems likely that even more such proposals will come in the future.

### Existing solutions aren't working anymore

Some new syntax features can be designed by using existing keywords. However, there are very few remaining reserved JavaScript keywords to ascribe meaning to--`enum` may be the only one that's not used at all.

#### Contextual keywords

In the past, we've dealt with that issue using "contextual keywords"--tokens which are keywords in one context, while remaining a variable name in other contexts. However, we're running out of space and energy to keep going with contextual keywords.

Recently added new contextual keywords like `let` and `async` created a combination of additional lookahead (with corresponding difficulty in maintaining parser efficiency) and surprising edge cases where they are treated as a variable instead. Even in cases that work out relatively cleanly, like `await` and `yield`, it's somewhat weird to have to look at the immediately enclosing function to tell whether it's a keyword or a variable (and there are still some non-trivial edge cases, e.g., with nested arrow functions). And the edge cases often result in differences between JavaScript implementations that take time to iron out.

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

It might be notable that it's hard to find historical examples of this technique, but it's all over the place in proposals! Repurposing existing keywords for something which is semantically fairly different risks being confusing for programmers. The JavaScript learner could ask, "but what does the `try` keyword actually do?" when the answer is "it's repurposed for multiple things which are only loosely related". That might not be such a good experience, even if it leads to something which is syntactically unambiguous and easy to parse without weird edge cases.

### Why not runtime decorators?

Many of these constructs need to run ahead of time, visible during parsing and static semantics. For example:
- Async functions and generator functions must be parse-able as such so that `await` or `yield` (respectively) are treated as keywords within their body.
- Syntactic tail call sites must be statically known to avoid a performance regression in existing code (imagine an extra lookup and conditional per call!).
- Resource usage blocks must take a block in a built-in way, otherwise there would be a requirement for anonymous functions which allow break, return and continue based on the *enclosing block*, which leads to serious implementation difficulties.
- Module import variants must be known at parse time, since modules are imported before any code in the module runs.

For this reason, class decorators or follow-on proposals based on function calls at runtime would not be suitable for many of the use cases that decorator-like syntax is proposed for.

## The idea of this proposal

Let's see if we can reduce the syntax variation of new proposals by introducing a new pattern which can be flexibly used in many of these different potential future language features: to modify an existing construct with a new feature, prepend it with a decorator-like syntax `@[decorator]`.

This proposal would not give any semantics to the construct `@[decorator]`. Instead, individual follow-on proposals would give meaning to that construct, both based on the position and based on the `decorator`--individual proposals are encouraged to give meaning to only a fixed set of identifiers (often one) that begin the expression `decorator`. This proposal just reserves the syntax space (ensuring it continues to lead to a SyntaxError) and suggests the possible pattern for future proposals.

### Credits for this proposal

Thanks to Ron Buckton for suggesting this particular syntax in [this decorators issue comment](https://github.com/tc39/proposal-decorators/issues/69#issuecomment-385609889).

Thanks to Leo Balter for raising the idea of using a variant of decorator syntax for syntactic tail calls, which was really helpful for getting me thinking on this path.

Thanks to Yehuda Katz for hia insight in bringing decorators to JavaScript and vision to use them as a basis for modifying all soets of syntactic constructs, and for discussions and reviews of this proposal.

Thanks to Waldemar Horwat, Kevin Gibbons, Mark Miller, Leo Balter, Dave Herman and more in TC39 for raising the urgency and priority of limiting the growth of syntactic complexity in JavaScript.

Thanks to Allen Wirfs-Brock for introducing the idea of reserving a syntactic space, where he proposed `keyword.pseudoproperty`, in the vein of `new.target` and `function.sent`.

Thanks to various reviewers for expressing confusion at the previous `@scope: expressionLike` syntax, leading to the change to this proposal's `@[expressionLike]` syntax. Note that this syntax proposal is still very early and should be considered up for discussion and unstable.

### Intuition for decorator syntax

From the perspective of a programmer who uses decorators, and doesn't dive into implementing their own, decorators are a construct for modifying the behavior of the decorated code. Decorators, including special decorator-like syntax which has `[]` around them, are doing just that. The `@` is a hint that this modification is going on, and what follows the decorator describes what's being modified.

### Related usage in other languages

Decorator-like constructs do different things in different languages; it's far from unusual to have built-in decorators which have magic powers, even alongside a rich language with various metaprogramming constructs.
- Swift [Attributes](https://docs.swift.org/swift-book/ReferenceManual/Attributes.html) provide a fixed set of modifiers--it is purely a namespace for built-in constructs using familar `@attribute`-style syntax.
- Rust [Attributes](https://doc.rust-lang.org/reference/attributes.html) provide a built-in set of modifiers, plus a `#[derive(XYZ)]` attribute for a developer-provided `XYZ`, where you can define your own custom `derive` behavior for a trait.
- C# [Attributes](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/concepts/attributes/) include (optional) "targets" before a `:`, similarly to this proposal. Several built-in attributes are provided which control fundamental things such as the loading, security, and JIT behaviors attached to code.

### Benefits of a unified syntax for modification

The goal here is not to encourage an explosion of new modifiers that everyone has to learn. But, when new modifiers do turn out to be needed in the evolution of the language, JavaScript developers should not have to learn a new set of syntactic rules to govern it, which are specific to that particular modifier.

## Applications

This section contains some extremely early ideas (not necessarily reviewed by the proposal champions!) for how some TC39 proposals could make use of the reserved decorator-like syntax proposed in this document.

### Auto-bound methods

Many of the use-cases for decorators have to do with various ways to "auto-bind" methods, that is, make it so that `this.method` refers to `this.method.bind(this)`. Class fields permit a pattern for using arrow functions in initializers, but this causes runtime overhead by pre-allocating the bound method per instance, when a bound method may not be needed. The decorator-based patterns can make that allocation lazy, but still, a bound method needs to be created each time a method is used, even if it's called immediately.

As a built-in feature, the runtime may be able to eliminate the creation of the bound method in the cases where it's called immediately, creating and caching it only when needed. It also may be more efficient to run during startup if it's built-in to the IC system rather than calling out to a custom getter (though in optimized code, the distinction may not exist).

<table><tr><th>Usage style<th>As decorator-like syntax
<tr><td>On a specific method<td>

```js
class C {
  @[autobind]
  method() { }
}
```
<tr><td>On all methods in a class<td>

```js
@[autobind]
class C {
  method() { }
}
```
</table>

### Explicit resource management

The [explicit resource management](https://github.com/tc39/proposal-using-statement) proposal creates a protocol and convenient syntax for disposing of resources after they are used in a block of code.

<table><tr><th>Usage mode<th>Syntax<th>As a decorator-like syntax
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
@[resource(file)] {
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
  @[resource]
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
  @[resource(file)];
  file.doStuff();
}
```
</table>

In this case, the decorator-like syntax `@[resource]` can either decorate a block, a variable declaration, or an empty statement, but serves an intuitively similar purpose in all cases.

### Module import variants

The [asset references](https://github.com/sebmarkbage/ecmascript-asset-references) introduces a new import-like construct, with the `asset` contextual keyword. This keyword faces similar problems to other contextual keywords, with the need for cover grammars, ASI hazards, and a thing that looks like `import` but is subtlely different syntactically.

Similarly, there have been various ideas about providing CORS or SRI-related metadata inline with module imports. Further thoughts about hashing have led to the refinement that signatures or hashes are better kept out-of-line (to avoid having to rehash/resign up the whole tree because a dependency changes), but there could be other kinds of metadata needed to pass into an import.

<table><tr><th>Proposal<th>Syntax<th>As decorator-like syntax
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
@[asset]
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
@[options({ option: "value", option2: "value2" })]
import Foo from "./foo.mjs";

```
</table>

### Syntactic tail calls

ES2015's implicit proper tail calls (PTC, also referred to as "tail call optimization") has only been shipping in JavaScriptCore (Safari) and not other JavaScript engines used in web browsers because of an unresolved disagreement about whether it's OK to have tail call semantics be opted into by default.

The ["Syntactic Tail Calls"](https://github.com/tc39/proposal-ptc-syntax/) (STC) proposal adds explicit syntax for tail calls. However, this proposal uses the syntax `return continue`, which not everyone is happy about. Decorator-like syntax could be used for an alternative syntax design.

<table><tr><th>Variant<th>Syntax<th>As decorator-like syntax
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

  return @[tailcall] factorial(n - 1, acc * n)
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
@[tailcall]
function factorial(n, acc = 1) {
  if (n === 1) {
    return acc;
  }

  return factorial(n - 1, acc * n)
}
```
</table>

In both cases, the `@[tailcall]` decorator-like syntax would be an assertion that the marked code contains a tail call, and the semantics the affected callsites would be treated as such. The placement is determined by the programmer, depending on their programming style and intuition. Decorating a callsite is a more specific assertion, whereas decorating the whole function can be convenient when there may be many tail call sites, and it would be repetitive to mark each one.

### Variants on classes

Different kinds of syntaxes have been tossed around for "defensible classes", "typed objects", "value types", etc. These other kinds of classes logically relate to classes somehow (they have methods, fields, may use private, etc), but have somewhat different semantics. Because these constructs are intended to be reliable, it makes sense to have a solid, built-in way to trigger them, and one way is with dedicated syntax.

Rather than introduce various kinds of contextual keywords, we could use decorator-like syntax for this sort of case:

<table><tr><th>Proposal<th>Syntax<th>As decorator-like syntax
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
@[defensible]
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
@[typed]
class Point {
  @[type(float64)]
  x;
  @[type(float64)]
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
@[value]
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


### Function.prototype.toString censorship

<table><tr><th><a href="https://github.com/domenic/proposal-function-prototype-tostring-censorship">Current proposal</a> form<th>As decorator-like syntax
<tr><td>

```js
"use no Function.prototype.toString";
function f() { }
function g() { }
```
<td>

```js
@[noFunctionPrototypeToString];
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
@[noFunctionPrototypeToString]
function h() { }
```
</table>

There is a particular question about how "deep" should these decorator syntaxes go--should such a construct be able to apply recursively, into nested constructs, e.g., inner functions? Although user-level programmatic decorators may never get that ability, it's unclear why decorator-like syntax should not be able to have those semantics. See [previous discussion](https://github.com/domenic/proposal-function-prototype-tostring-censorship/issues/13).

Note that the addition of this syntax would not meet the "backwards compatibility" goal of the Function.prototype.toString proposal--it would be a syntax error to use it in browsers that do not support it; see [this issue](https://github.com/domenic/proposal-function-prototype-tostring-censorship/issues/9) for discussion.

### Operator overloading declarations

<table><tr><th><a href="https://github.com/littledan/proposal-operator-overloading/">Current proposal</a> form<th>As decorator-like syntax
<tr><td>

```js
with operators from Scalar, Vector;
function f() { return scalar + vector; }
```
<td>

```js
@[operators(Scalar, Vector)];
function f() { return scalar + vector; }
```
</table>


### Custom code transforms

Many people are experimenting with using various kinds of code generation in JavaScript, e.g., with [Babel](https://babeljs.io/) plugins. Babel's parser only permits standards-track syntax (plus JSX and type systems) to be parsed, so transforms end up giving new semantics to existing syntax. Sometimes, the semantics provided can be considered in line with standard JavaScript, and sometimes, they may be somewhat distinct. Because there's no syntactic place to "opt in" to other semantics, new interpretations of existing constructs are the only option.

There's a composability issue here: If these transforms are done globally, rather than specifically to just constructs that need them, the code base and the transform sort of go together, and it might not be possible to develop two different types of code together, if the transforms need to be used on the file as a whole.

To resolve this, the decorator-like syntax `@[expression]` construct could be used to specifically call for a transform on a particular piece of code which is affected.  These would all be syntax errors executed in JavaScript without transformation by tools (since only known scopes with specification-defined behavior are accepted), without including their own custom syntax.

#### Managing scope names

There is some risk that different non-standard scope names may overlap and conflict, much like different libraries that add properties of the same name to the global object may overlap and conflict. Unlike in libraries, there's no clear way to "import" these constructs to disambiguate (though one could imagine an analogous, tools-only "import type"-style construct for decorator-like syntax); decorator-like syntax isn't based on lexical scoping. This would be unfortunate, but at least it would be a limited-impact conflict (to those particular constructs).

Some mitigation strategies to reduce the risks of overlaps:
- By convention, scopes which are project-specific could use related names, such as `@[ember.identifier]` or `@[ts.identifier]`, could be used to call out to tool- or project-specific transforms, allowing these plugins to be used together in the same code bases.
- We could build a central place to share information about what kinds of scope names are used by projects; when choosing a new scope name, people can consult this list and choose one not included, and then note their choice on the list. Participation would be optional but beneficial to participants. This work could take place in the [JS shared interfaces repository](https://github.com/littledan/js-shared-interfaces/).

## Syntactic details

The grammar of a decorator-like syntax is based on ordinary decorators:

```
  DecoratorList[Yield, Await]:
    DecoratorList[?Yield, ?Await]opt Decorator[?Yield, ?Await]
    DecoratorList[?Yield, ?Await]opt DecoratorLike[?Yield, ?Await]

  Decorator[Yield, Await]:
    @ DecoratorMemberExpression[?Yield, ?Await]
    @ DecoratorCallExpression[?Yield, ?Await]

  DecoratorLike[Yield, Await]:
    @ [ DecoratorMemberExpression[?Yield, ?Await] ]
    @ [ DecoratorCallExpression[?Yield, ?Await] ]

  DecoratorMemberExpression[Yield, Await]:
    IdentifierReference[?Yield, ?Await]
    DecoratorMemberExpression[?Yield, ?Await] . IdentifierName
    ( Expression[+In, ?Yield, ?Await] )

  DecoratorCallExpression[Yield, Await]:
    DecoratorMemberExpression Arguments[?Yield, ?Await]
```

`DecoratorList`s could be potentially used in various contexts, including preceding the syntactic productions `Statement` and `MemberExpression`, including (some of these may only be available for decorator-like syntax and not ordinary decorators):
- Empty statement (just a semicolon)
- Class declarations
- Function declarations (including async and generator)
- `if` statements, loops, `return` statements, etc
- Variable declarations
- Object literals
- Array literals
- Function calls
- Arrow functions

Initially, "Static Semantics: Early Errors" rules are used to maintain a syntax error in all cases of using decorator-like syntax, and all cases where decorators exist that aren't supported by the [class decorators](https://github.com/tc39/proposal-decorators/) proposal. Future proposals may chip away at this space, defining various subsets, while leaving the rest in tact for further proposals. The encouraged path is to never give an interpretation to the entire `@[decorator]` space, and instead chose individual `decorator`s to define, to leave space open for later.

### ASI hazard for `EmptyStatement`

One use case for decorator-like syntax is a statement-like pragma, which covers the rest of the program, as in `@[operators(Vector)];` above. In this case, the semicolon is mandatory, otherwise the immediately following construct will be decorated. I believe that it should be OK to require a semicolon in a fixed construct like this.

An alternative is to say that the `IdentifierName` determines whether it's an `EmptyStatement` decorator or one which takes a larger syntactic construct afterwards. This option is dispreferred, as it weakens the usability of decorator-like syntax as an extension point for tools (those tools would have to pass in this parameter to the parser).

## Alternatives considered

Aside from the current strategies cited earlier, there has been some discussion of other syntaxes for certain modified constructs:
- Instead of the `@[decorator]` syntax, we could use the more abbreviated syntax `@@decorator`. The square brackets are used to emphasize the mode switch: the thing inside of the brackets are not treated as an expression, but rather have a special treatment at parse time. It's also beneficial to have an end token, to enable decorating more types of expressions, as well as simplifying the parsing of arrow functions with decorator-like syntax.
- For value object literals, the syntax `#{ }` could be used, rather than something like `@[record] { }`, and similarly, `#[ ]` vs `@[tuple] [ ]`.
- Keywords following `@` are disallowed, so constructs like `@const` for defensible classes is a possible area for expansion, though more limited.
- Decorators for certain constructs which don't currently permit decorators  could be interpreted as decorator-like syntax by default, without a scope prefix (though this cuts off a potentially important evolutionary path where we later do have semantics for procedural decorators).
- A previous version of this proposal used `@prefix: expressionLike` syntax, but that seemed to be confusing.

There is no contradiction between continuing to define certain specialized syntaxes and setting aside a built-in syntax fallback. We can initially prototype features using scoped decorator-like syntax, and later consider these more terse syntaxes for features where we decide it makes more sense.

## Next steps

This proposal doesn't need to be at any particular stage, but if it's a direction that looks generally attractive, we might consider the following:
- Ensure that the placement of decorators doesn't close off any particular future paths
- When proposing another syntactic construct for JavaScript, consider whether it could use decorator-like syntax.
- Adding parser support for decorator-like syntax (and general metaproperty syntax) into tools like Babel, so that transform plugins can experiment with applying semantics to it.

### Implications for the class decorators proposal

- Class decorators should probably come after `export`, as we may want decorator-like syntax to modify `export` declarations themselves, even if it seems implausible to have a procedural decorator modifying `export`.
- There are many use cases for decorator-like syntax modifying deeply nested constructs, so we should make sure to build a clear intuition for this among decorators users. In the context of such an intuition, it makes sense for ordinary class decorators to be able to access private class elements.
