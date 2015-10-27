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
function curry<T, U, V>(f: (t: T, u: U) => V): (a:T) => (b:U) => V {
    return a => b => f(a, b);
}
```

However, a variadic version is easy to write in Javascript but cannot be given a type in TypeScript:

```js
function curry(f) {
    return ...a => ...b => f(...a, ...b);
}
```

Here's an example of using variadic kinds to type `curry`:

```ts
function curry<...T,...U,V>(f: (...ts: [...T, ...U]) => V): (...as:...T) => (...bs:...U) => V {
    return ...a => ...b => f(...a, ...b);
}
```

The syntax for variadic tuple types that I use here matches the spread and rest syntax used for values in Javascript.
This is easier to learn but might make it harder to distinguish type annotations from value expressions.
Similarly, the syntax for concatenating

Open questions:
Semantics of matching concatenated types -- it's fine to have a concatenated result, but what does it mean to match a concatenated parameter?

To address question 3, let's look at an example call to `curry`:

```ts
function f(n: number, m: number, s: string, c: string): [number, number, string, string] {
    return [n,m,s,c];
}
let [n,m,s,c] = curry(f)(1, 2)('foo', 'x');
```

TODO: Example for object-kinded too.

## Tuple-kinded type variables

## Tuple-kinded type operators

## Object-kinded type variables

## Object-kinded type operators

## Type checking rules

## Equivalence of parameter lists and tuple/object kinds

## Examples

1. concat
2. apply
3. curry
4. pipe/compose
5. decorators

## Implementation concerns

## Missed parts, open questions and future work

1. Does the call site need type annotations? C# does even with fixed-arity higher-order functions.

