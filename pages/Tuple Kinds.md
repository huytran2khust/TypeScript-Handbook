# Variadic Kinds 

## Give Specific Types to Variadic Functions

This proposal lets Typescript give types to higher-order functions that take a variable number of parameters.
Functions like this include `concat`, `apply`, `curry`, `compose` and almost any decorator that wraps a function.
In Javascript, these higher-order functions are expected to accept variadic functionsas arguments.
With the ES2015 and ES2017 standards, this use will become even more common as programmers start using spread arguments and rest parameters for both arrays and objects.
This proposal addresses these use cases with a single, very general typing strategy based on higher-order kinds.

## Preview example with `curry`

`curry` for functions with two arguments is simple to write in Javascript and Typescript:

```js
function curry(f, a) {
    return b => f(a, b);
}
```

and in Typescript with type annotations:

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

Here's an example of using variadic kinds from this proposal to type `curry`:

```ts
function curry<...T,...U,V>(f: (...ts: [...T, ...U]) => V, ...as:...T): (...bs:...U) => V {
    return ...b => f(...a, ...b);
}
```

The syntax for variadic tuple types that I use here matches the spread and rest syntax used for values in Javascript.
This is easier to learn but might make it harder to distinguish type annotations from value expressions.
Similarly, the syntax for concatenating looks like tuple construction, even though it's really concatenation of two tuple types.

Now let's look at an example call to `curry`:

```ts
function f(n: number, m: number, s: string, c: string): [number, number, string, string] {
    return [n,m,s,c];
}
let [n,m,s,c] = curry(f, 1, 2)('foo', 'x');
let [n,m,s,c] = curry(f, 1, 2, 'foo', 'x')();
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
...T = [number, number, string, string]
...U = []
```

## Syntax

The syntax of a variadic kind variable is `...T` where *T* is an identifier that is by convention a single upper-case letter, or `T` followed by a `PascalCase` identifier.
Variadic kind variables can be used in a number of syntactic contexts:

Variadic kinds can be bound in the usual location for type parameter binding, including functions and classes:

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
    // note that U is constrained to [string,string] in this function
    let us: ...U = makeTuple('hello', 'world');
    return [...ts, ...us];
}
```

Variadic kind variables, like type variables, are quite opaque.
They do have one operation, unlike type variables.
They can be concatenated with other kinds or with actual tuples.
The syntax used for this is identical to the tuple-spreading syntax, but in type annotation location:

```ts
let t1: [...T,...U] = [...ts,...uProducer<...U>()];
let t2: [...T,string,string,...U,number] = [...ts,'foo','bar',...uProducer<...U>(),12];
```

Tuple types are instances of variadic kinds, so they continue to appear wherever type annotations were previously allowed:

```ts
function f<...T>(ts:...T): [...T,string,string] { 
    // note the type of `us` could have been inferred here
    let us: [string,string] = makeTuple('hello', 'world');
    return [...ts, ...us];
}

let tuple: [number, string] = [1,'foo'];
f<[number,string]>(tuple);
```

## Semantics

A variadic kind variable represents a tuple type of any length. 
Since it represents a set of types, we use the term 'kind' to refer to it, following its use in type theory.
Because the set of types it represents is tuples of any length, we qualify 'kind' with 'variadic'.

Therefore, declaring a variable of variadic tuple kind allows it to take on any *single* tuple type.
Like type variables, kind variables can only be declared as parameters to functions, classes, etc, which then allows them to be used inside the body:

```ts
function f<...T>(): ...T {
    let a: ...T;
}
```

Calling a function with arguments typed as a variadic kind will assign a specific tuple type to the kind:

```ts
f([1,2,"foo"]);
```

Assigns the tuple type `...T=[number,number,string]`...T`. 
So in this application of `f`, `let a:...T` is instantiated as `let a:[number,number,string]`.
However, because the type of `a` is not known when the function is written, the elements of the tuple cannot be referenced in the body of the function.
Only creating a new tuple from `a` is allowed.
For example, new elements can be added to the tuple: 

```ts
function cons<H,...Tail>(head: H, tail: ...Tail): [H,...Tail] {
    return [head, ...tail];
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

Additionally, variadic kind variables can be inferred when concatenated with types:

```ts
function car<H,...Tail>(l: [H, ...Tail]): H {
    let [head, ...tail] = l;
    return head;
}
car([1, "foo", false]);
```

Here, the type of `l` is inferred as `[number, string, boolean]`. 
Then `H=number` and `...Tail=[string, boolean]`.

### Limits on type inference

Concatenated kinds cannot be inferred because the checker cannot guess where the boundary between two kinds should be:

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

Uncheckable dependencies between type arguments and the function body can arise, as in `rotate`:

```ts
function rotate(l:[...T, ...U], n: number): [...U, ...T] {
    let first: ...T = l.slice(0, n);
    let rest: ...U = l.slice(n);
    return [...rest, ...first];
}
rotate<[boolean, boolean, string], [string, number]>([true, true, 'none', 12', 'some'], 3);
```

This function can be typed, but there is a dependency between `n` and the kind variables: `n === ...T.length` must be true for the type to be correct.
I'm not sure whether this is code that should actually be allowed.

### Semantics on classes and interfaces

The semantics are the same on classes and interfaces.

TODO: There are probably some class-specific wrinkles in the semantics.

## Assignability between tuples and parameter lists

Tuple kinds can be used to give a type to rest arguments of functions inside their scope:

```ts
function apply<...T,U>(ap: (...args:...T) => U, args: ...T): U {
    return ap(...args);
}
function f(a: number, b: string) => string {
    return b + a;
}
apply(f, [1, 'foo']);
```

In this example, the parameter list of `f: (a: number, b:string) => string` must be assignable to the tuple type instantiated for the kind `...T`.
The tuple type that is inferred is `[number, string]`, which means that `(a: number, b: string) => string` must be assignable to `(...args: [number, string]) => string`.

As a side effect, function calls will be able to take advantage of this assignability by spreading tuples into rest parameters, even if the function doesn't have a tuple kind:

```ts
function g(a: number, ...b: [number, string]) {
    return a + b[0];
}
g(a, ...[12, 'foo']);
```

### Tuple types generated for optional and rest parameters

Since tuples can't represent optional parameters directly, when a function is assigned to a function parameter that is typed by a tuple kind, the generated tuple type is a union of tuple types.
Look at the type of `h` after it has been curried:

```ts
function curry<...T,...U,V>(cur: (...args:[...T,...U]) => V, ...ts:...T): (...us:...U) => V {
    return ...us => cur(...ts, ...us);
}
function h(a: number, b?:string): number {
}
let curried = curry(h, 12);
curried('foo'); // ok
curried(); // ok
```

Here `...T=([number] | [number, string])`, so `curried: ...([number] | [number, string]) => number` which can be called as you would expect. Unfortunately, this strategy does not work for rest parameters. These just get turned into arrays:

```ts
function i(a: number, b?: string, ...c: boolean[]): number {
}
let curried = curry(i, 12);
curried('foo', [true, false]);
curried([true, false]);
```

Here, `curried: ...([string, boolean[]] | [boolean[]]) => number`.
I think this could be supported if there were a special case for functions with a tuple rest parameter, where the last element of the tuple is an array.
In that case the function call would allow extra arguments of the correct type to match the array.
However, that seems too complex to be worthwhile.

## Extensions to the other parts of typescript

1. Typescript does not allow users to write an empty tuple type.
  However, this proposal requires variadic kinds to be bindable to a empty tuple.
  So Typescript will need to support empty tuples, even if only internally.

## Examples

Most of these examples are possible as fixed-argument functions in current Typescript, but with this proposal they can be written as variadic.
Some, like `cons` and `concat`, can be written for homogeneous arrays in current Typescript but can now be written for heteregoneous tuples using tuple kinds.
This follows typical Javascript practise more closely.

### Return a concatenated type

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

let start: [number,number] = [1,2]; // type annotation required here
cons(3, start); // == [3,1,2]
```

### Concatenated type as parameter

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

### Variadic functions as arguments

```ts
function apply<...T,U>(f: (...args:...T) => U, args: ...T): U {
    return f(...args);
}

function f(x: number, y: string) {
}
function g(x: number, y: string, z: string) {
}

apply(f, [1, 'foo']); // ok
apply(f, [1, 'foo', 'bar']); // too many arguments
apply(g, [1, 'foo', 'bar']); // ok
```

```ts
function curry<...T,...U,V>(f: (...args:[...T,...U]) => V, ...ts:...T): (...us: ...U) => V {
    return us => f(...ts, ...us);
}
let h: (...us: [string, string]) = curry(f, 1);
let i: (s: string, t: string) = curry(f, 2);
h('hello', 'world');
```

```ts
function compose<...T,U,V>(f: (u:U) => U, g: (ts:...T) => V): (args: ...T) => V {
    return ...args => f(g(...args));
}
function first(x: number, y: number): string {
}
function second(s: string) {
}
let j: (x: number, y: number) => void = compose(second, first);
j(1, 2);

```

TODO: Could `f` return `...U` instead of `U`?

### Decorators

```ts
function logged<...T,U>(target, name, descriptor: { value: (...T) => U }) {
    let method = descriptor.value;
    descriptor.value = function (...args: ...T): U {
        console.log(args);
        method.apply(this, args);
    }
}
```

## Open questions

1. Does the tuple-to-parameter-list assignability story hold up? It's especially shaky around optional and rest parameters.
2. Specifically, should rest parameters be special cased to retain their nice calling syntax, even when they are generated from a tuple type? (In this proposal, functions typed by a tuple kind have to pass arrays to their rest parameters, they can't have extra parameters.)
