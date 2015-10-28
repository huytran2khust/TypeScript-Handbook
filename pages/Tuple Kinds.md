# Tuple and Object Kinds for Typing Rest Parameters

Or as an impenetrable noun compound, "variadic object kinds".

The basic objective of this proposal is to let Typescript users type higher-order functions that take an variable number of optionally named parameters.
Example of functions like this include any decorator, `concat`, `apply`, `curry`, `compose`/`pipe`.
Essentially, any higher-order function that could be written in a functional language with fixed arity must be variable arity in TypeScript in order to support the variety of Javascript uses.
This includes the ES2015 and ES2017 standards, which include spread arguments and rest parameters for both arrays and objects.

Currently, Typescript does not support the ES2015 spec for spread/rest in argument lists precisely because it has no way to type most combinations of spread arguments with no rest arguments.
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

This function can be typed, but there is a depedency between `n` and the kind variables: `n === ...T.length` must be true for the type to be correct.
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

Variadic kinds can be bound in the usual location for type variable binding:

```ts
function f<...T,...U>() {}
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
However, like type variables, kind variables can only be declared by functions:

```ts
function f<...T>(): ...T {
    let a: ...T;
}
```

### Type checking and type inference

Concatenations of kinds cannot be used for type inference. However, you can still annotate the types explicitly.

## Equivalence of parameter lists and tuple/object kinds

Maybe. I'm not sure this is actually the case.

## Examples

1. concat
2. apply
3. curry
4. pipe/compose
5. decorators

## Implementation concerns

## Missed parts, open questions and future work

1. Does the call site need type annotations? C# does even with fixed-arity higher-order functions.
