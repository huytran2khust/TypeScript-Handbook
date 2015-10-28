# Variadic Kinds 

## Give Specific Types to Variadic Functions

This proposal let Typescript users type higher-order functions that take a variable number of parameters.
Example of functions like this include any `concat`, `apply`, `curry`, `compose`/`pipe` or almost any wrapping decorator.
Essentially, any higher-order function that could be written in a functional language with fixed arity must be variable arity in TypeScript in order to support the variety of Javascript uses.
With the ES2015 and ES2017 standards, this use will become even easier as programs start using spread arguments and rest parameters for both arrays and objects.

Currently, Typescript does not support the entire ES2015 spec for spread/rest in argument lists precisely because it has no way to type most spread arguments that do not match a rest argument.
And its support for tuples does not capture nearly all the tuple-like patterns that have been used in Javascript for a long time.
And the ES2017 spec has a stage 2 proposal for spread/rest of object types which 
This proposal addresses all these use cases with a single, very general typing strategy based on higher-order kinds.

But higher-order kinds in a way that has not been attempted before. 
Most languages with higher-order kinds and types at all simply avoid spread arguments. 

## Similar work in related languages

Apparently Flow has an undocumented `Either` type of types already. I haven't gone looking for any discussion yet.
Nobody has a full solution for this as far as we know. 
Typed languages normally avoid it (or simplify), dynamic languages don't care, and progressively typed languages have punted so far (I have checked Racket, Strongtalk and [Common Lisp](http://www.lispworks.com/documentation/HyperSpec/Issues/iss178_w.htm) so far).

## Preview example with `curry` and decorator

`curry` in Haskell is simple because it only takes two arguments:

```hs
curry :: ((a,b) -> c) -> a -> b -> c
curry f a b = f (a,b)
```

It's equally easy in Typescript with only two arguments:

```ts
function curry<T, U, V>(f: (t: T, u: U) => V, a:T): (b:U) => V {
    return b => f(a, b);
}
```

However, a variadic version is easy to write in Javascript but cannot be given a type in TypeScript:

```js
function curry(f, ...a) {
    return ...b => f(...a, ...b);
}
```

Here's an example of using variadic kinds to type `curry`:

```ts
function curry<...T,...U,V>(f: (...ts: [...T, ...U]) => V, ...as:...T): (...bs:...U) => V {
    return ...b => f(...a, ...b);
}
```

The syntax for variadic tuple types that I use here matches the spread and rest syntax used for values in Javascript.
This is easier to learn but might make it harder to distinguish type annotations from value expressions.
Similarly, the syntax for concatenating looks like tuple construction, even though it's really concatenation of two type lists.

To address this question, let's look at an example call to `curry`:

```ts
function f(n: number, m: number, s: string, c: string): [number, number, string, string] {
    return [n,m,s,c];
}
let [n,m,s,c] = curry(f, 1, 2)('foo', 'x');
let [n,m,s,c] = curry(f, 1, 2, 'foo')('x');
```

In the first call, 

```
V = [number, number, string, string]
...T = [number, number]
...U = [string, string]
```

In the second call, 

```
V = [number, number, string, string]
...T = [number, number, string]]
...U = [string]
```

Note that non-rest parameters can be typed as tuple kinds.
Note that concatenations of variadic kinds cannot be used for type inference, such as in `rotate`:

```js
function rotate(l, n) {
    let first = l.slice(0, n);
    let rest = l.slice(n);
    return [...rest, ...first];
}
rotate([true, true, 'none', 12, 'some'], 3); // returns [12, 'some', true, true, 'none']
```

```ts
function rotate(l:[...T, ...U], n: number): [...U, ...T] {
    let first: ...T = l.slice(0, n);
    let rest: ...U = l.slice(n);
    return [...rest, ...first];
}
rotate<...[boolean, boolean, string], ...[string, number]>([true, true, 'none', 12', 'some'], 3);
```

This function can be typed, but there is a dependency between `n` and the kind variables: `n === ...T.length` must be true for the type to be correct.
 I'm not sure whether this is code that should actually be supported.
 Even if it is, a verbose type annotation is a worthwhile price to pay to handle homogenous lists in Typescript's fairly buttoned-down type system.

The previous examples show tuples used for variadic kinds. 
Note that, if JavaScript supported named arguments, then objects could also be expected to be spread into function calls, and Typescript would also need variadic object kinds.
Currently [a stage 2 ECMAScript proposal](https://github.com/sebmarkbage/ecmascript-rest-spread) supports object spreading, but only within object destructuring contexts. Here is an example:
In Javascript:

```js
function namedCurry(f, obj1) {
    return obj2 => f({...obj1, ...obj2});
}
function f({a, b, c, d}) {
}
namedCurry(f, {a: 12, b: 34})({c: 56, d: 78});
```

And now with types in TypeScript:

```ts
interface FirstHalf {
    a: string;
    b: number;
}
interface SecondHalf {
    c: string;
    d: boolean;
}
interface Total extends FirstHalf, SecondHalf {
}
function namedCurry(f: Total => void, obj1: FirstHalf): SecondHalf => void   {
    return obj2 => f({...obj1, ...obj2});
}
```

So, actually this example doesn't need kinds. That's because named parameters don't exist in Ecmascript 2015: the `arguments` array is an array, not an object. 

## Syntax

The syntax of a variadic kind variable is `...T` where *T* is an identifier that is by convention a single upper-case letter, or `T` followed by a `PascalCase` identifier.
Variadic kind variables can be used in a number of syntactic contexts:

### Variadic kind variables

Variadic kinds can be bound in the usual location for type variable binding, including functions and classes:

```ts
function f<...T,...U>() {}
}
class C<...T> {
}
```

And they can be referenced in any type annotation location:

```ts
function makeTuple<...T>(ts:...T): ...T {
    return ts;
}
function f<...T,...U>(ts:...T): [...T,...U] {
    let us: ...U = makeTuple('hello', 'world'); // note that U is constrained to [string,string] here
    return [...ts, ...us];
}
```

Tuples are instances of variadic kinds, so they continue to appear wherever tuple type annotations were previously allowed:

```ts
function f<...T>(ts:...T): [...T,string,string] {
    let us: [string,string] = makeTuple('hello', 'world'); // note the annotation can be inferred here
    return [...ts, ...us];
}

let tuple: [number, string] = [1,'foo'];
f<[number,string],[string,string]>(tuple);
```

### Variadic kind operations

Variadic kind variables, like type variables, are quite opaque and are not present at runtime.
They do have one operation, unlike type variables.
They can be concatenated with other kinds or with actual tuples.
The syntax used for this is identical to the tuple-spreading syntax, but in type annotation location:

```ts
let t1: [...T,...U] = [...ts,...uProducer<...U>()];
let t2: [...T,string,string,...U,number] = [...ts,'foo','bar',...uProducer<...U>(),12];
```

## Semantics

A variadic kind variable represents a tuple type of any length. 
Since it represents a set of types, we use the term 'kind' to refer to it, following its use in type theory.
Because the set of types it represents is tuples of any length, we qualify 'kind' with 'variadic'.

Therefore, declaring a variable of variadic tuple kind allows it to take on any *single* tuple type.
Like type variables, kind variables can only be declared by functions, which then allows them to be used inside the body of the function:

```ts
function f<...T>(): ...T {
    let a: ...T;
}
```

Calling a function with arguments typed as a variadic kind will assign a specific tuple type to the kind:

```ts
f([1,2,"foo"]);
```

Assigns the tuple type `[number,number,string]` to `...T`. 
So in this application of `f`, `let a:...T` is equivalent to `let a:[number,number,string]`.
However, because the type of `a` is not known when the function is written, the elements of the tuple cannot be referenced.
Only concatenation will work.
For example, new elements can be added to the tuple: 

```ts
function cons<H,...Tail>(car: H, cdr: ...Tail): [H,...Tail] {
    return [car, ...cdr];
}
let l: [number, string, string, boolean]; 
l = cons(1, cons("foo", ["baz", false]));
```

Like type variables, variadic kind variables can usually be inferred.
The calls to `cons` could have been annotated:

```ts
l = cons<number,[string,string,boolean]>(1, cons<string,[string,boolean]>("foo", ["baz", false]));
```

For example, `cons` must infer two variables, a type *H* and a kind *...Tail*. 
In the innermost call, `cons("foo", ["baz", false])`, `H=string` and `...Tail=[string,boolean]`. 
In the outermost call, `H=number` and `...Tail=[string, string, boolean]`.
The types assigned to *...Tail* are obtained by typing list literals as tuples -- variables of a tuple type can also be used:

```ts
let tail: [number, boolean] = ["baz", false];
let l = cons(1, cons("foo", tail));
```

### Limits on type inference

Note that concatenated kinds cannot be inferred because the checker cannot guess where the boundary between two kinds should be:

```ts
function twoKinds<...T,...U>(total: [...T,string,...U]) {
}
twoKinds("an", "ambiguous", "call", "to", "twoKinds")
```

The checker cannot decide whether to assign 

1. `...T = [string,string,string], ...U = [string]`
1. `...T = [string,string], ...U = [string,string]`
1. `...T = [string], ...U = [string,string,string]`

Some unambiguous calls are a casualty of this restriction:

```ts
twoKinds(1, "unambiguous", 12); // but still needs an annotation!
```

The solution is to add type annotations:

```ts
twoKinds<[string,string],[string,string]>("an", "ambiguous", "call", "to", "twoKinds");
twoKinds<[number],[number]>(1, "unambiguous", 12);
```

### Extensions of tuple types

Typescript does not allow users to write an empty tuple type.
However, this proposal requires variadic kinds to be bound to a empty tuple.
So Typescript will need to support empty tuples, even if only internally.

### Semantics on classes and interfaces

The semantics are the same on classes and interfaces.

## Examples

Most of these examples are possible as fixed-argument functions in current Typescript, but with this proposal they can be written as variadic.
This follows typical Javascript practise more closely.

### cons/concat

```ts
function cons<H,...T>(head: H, tail:...T): [H, ...T] {
    return [head, ...tail];
}
function concat<...T,...U>(first: ...T, ...second: ...U): [...T, ...U] {
    return [...first, ...second];
}
cons(1, ["foo", false]); // === [1, "foo", false]
concat(['a', true], 1, 'b'); // === ['a', true, 1, 'b']
concat(['a', true]); // === ['a', true, 1, 'b']
```

### apply

```ts
function apply<...T,U>(f: (args:...T) => U, args: ...T): U {
    return f(args);
}
```

### curry

```ts
function curry<...T,...U,V>(f: (...args:[...T,...U]) => V, ts:...T): (us: ...U) => V {
    return us => f(...ts, ...us);
}
```

### pipe/compose

```ts
function pipe<...T,U,V>(f: (ts:...T) => U, g: (u:U) => V): (args: ...T) => V {
    return ...args => g(f(...args));
}
```

```ts
function compose<...T,U,V>(f: (u:U) => U, g: (ts:...T) => V): (args: ...T) => V {
    return ...args => f(g(...args));
}
```

TODO: Could `f` return `...U` instead of `U`?

### decorators

```ts
function logged<...T,U>(target, name, descriptor: { value: (...T) => U }) {
    let method = descriptor.value;

    descriptor.value = function (...args: ...T): U {
      console.log(args);
      method.apply(this, args);
    }
}
```

### Tuple copy implemented as `Array.slice()`

A function-based `slice` is simple to write:

```ts
slice<...T>(tuple: ...T): ...T;
```

A class-based `slice` requires the containing class to bind a variadic kind variable:

```ts
interface Tuple<...T> {
    slice(): ...T;
}
```

### Spread arguments that don't directly match a rest parameter

That is, `car` and `cdr`:

```ts
function car<H,...T>(l: [H,...T]): H {
    let [head, ...tail] = l;
    return head;
}
function cdr<H,...T>(l: [H,...T]): ...T {
    let [head, ...tail] = l;
    return ...tail;
}

cdr(["foo", 1, 2]); // => [1,2]
car(["foo", 1, 2]); // => "foo"
```