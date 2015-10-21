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

Apparently haxe and Flow might have done something? Maybe?
Nobody has a full solution for this as far as we know. 
Typed languages normally avoid it (or simplify), dynamic languages don't care, and progressively typed languages have punted so far (I have checked Racket, Strongtalk and [Common Lisp](http://www.lispworks.com/documentation/HyperSpec/Issues/iss178_w.htm) so far).
Flow has an ad-hoc solution that is undocumented so far. 
haxe might have thought about it too?

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

However, a variadic version is easy to write in Javascript but difficult to type in TypeScript:

```js
function curry(f) {
    return ...a => ...b => f(...a, ...b);
}
```

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

