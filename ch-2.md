## Managing State in Functional Programming

How does FP manage state when their are neither mutations nor reassignments? Let us take a little detour first, so that we are all on the same page before beginning to answer this question.

### Defining state

State is data that changes over the runtime of a program. In imperative programming you usually get state either by reassigning a variable

```javascript
let x = 1; // data
x = x + 1; // state
```
or by mutating a reference value:

```javascript
const o = {foo: true}; // data
o.foo = false; // state
```
### Variables and reassignments

Although the term variable is used in functional programming and math I prefer to avoid it because it regularly causes confusion: There are no variables in FP at least not those as they are understood in imperative programming. All we have at our disposal are immutable name bindings, that is you cannot reassign another value to an existing name. 

Reassignments are banned in functional programming since they would violate referential transparency:

```javascript
let = y = 2;
const sub2 = x => x – y

sub2(2); // 0
y = 3;
sub2(2); // -1
```
The same expression within the same scope must yield the same value no matter when the runtime decides to actually evaluate it. Referential transparency is a property of purity and was already discussed in a previous chapter of this course.

### References and mutations

The term referential transparency already dictates that the functional paradigm does not have a notion of references. References require the opposite namely referential identity, which allows certain values to be distinguished by their reference and thus by their location in memory. Such values constitute an identity:

```javascript
const o = {foo: true},
  p = {foo: true};

o !== p; // true
```
The expression above compares two references. In a referential transparent environment `o` and `p` are indistinguishable, because their values are exactly the same. While comparing two references in Javascript is possible it is still an impure operation. Only use it with great care. I will introduce a purely functional way to work with reference types in a later chapter.

Without references there is no identity and thus no reasonable way to mutate values anymore. The only alternative left is to simply create a new value whenever we need a modified one. So instead of a single value that changes over time we work with sequences of immutable values.

Creating new values is an expensive operation as soon as we have to deal with complex composite structures. Fortunately the functional world has developed persistent data structures that utilize structural sharing. With structural sharing only those parts of a composite value are actually copied that have been changed whereas the rest is simply reused. Purely functional data structures are covered in a later chapter of this course.

### The function call stack

If we want to work with sequences of distinct values we need a way to bind them to names and to distribute them throughout the code. However, as already mentioned we are not allowed to rebind new values to existing names in the same scope - or imperatively put: reassign a variable to a new value. How do we get out of this dilemma? Well, in FP it is all about functions. Let us just utilize their scopes.

Whenever we need a reassignment we just call a function with the desired value as its argument. Now if you squint hard enough you can think of immutable name bindings as variables, because the same name can hold various values provided it is declared in different function scopes. From this perspective a variable is just a name binding which exists in consecutive function call stack elements.

If the number of reassignments is static, i.e. it can be determined upfront, we can manually nest function calls:

```javascript
const scanSqr3 = w => xs => // A
  app(x => // B
    app(y => // B
      app(z => [x, y, z]) (sqr(y))) (sqr(x))) (sqr(w)); // B

const app = f => x => f(x); // auxiliary function

scanSqr3(2) ([]); // [4, 16, 256]
```
[run code](https://repl.it/repls/BouncySpottedCallback)

In dynamic scenarios we have to fall back to recursive solutions:

```javascript
const scanSqr = n => x => xs => // A
  n === 0
    ? xs
    : scanSqr(n - 1) (sqr(x)) (xs.concat(sqr(x))); // C

scanSqr(5) (2) ([]); // [4, 16, 256, 65536, 4294967296]
```
[run code](https://repl.it/repls/HatefulUrbanOpengroup)

The decisive point is that this approach scales. In a later chapter we will discuss how to structure large applications in a functional manner.

### Local bindings

While the above examples illustrate the fundamental concept of how state is managed in functional programming, they both have drawbacks we would like to avoid:

* we leak API by defining the empty array as a formal parameter (A)
* we have to create nested anonymous functions for each name binding (B)
* the same expression is evaluated twice for each recursive step (C)

We can simplify the above code by creating local bindings. Local bindings are just a special syntax to create name bindings within a scope. Under the hood they are nothing more than implicitly called nested anonymous functions. Javascript does not supply such bindings, but we can employ default parameters in a creative way to accomplish a similar effect:

```javascript
const _let = f => f();

const scanSqr3 = w =>
  _let((x = sqr(w), y = sqr(x), z = sqr(y)) =>
    [x, y, z]);

const scanSqr = n => x => xs => // (D)
  n === 0
    ? xs
    : _let((y = sqr(x)) => scanSqr(n - 1) (y) (xs.concat(y)));

scanSqr3(2); // [4, 16, 256]
scanSqr(5) (2) ([]); // [4, 16, 256, 65536, 4294967296]
```
[run code](https://repl.it/repls/LovingRegalParameter)

We have managed to improve the code considerably. However, we were not able to eliminate the last parameter (D) for the recursive function. It turns out that local bindings are just not enough for this kind of optimization. We would need fixed point combinators that allow anonymous recursion to achieve the desired behavior. Since we deal with Javascript we should not be dogmatic anyway, so let us declare an imperative function with brackets and explicit return statement, so that we can take advantage of an inner auxiliary function:

```javascript
const scanSqr = n => x => {
  const go = (xs, x_, n_) =>
    n_ === 0
      ? xs
      : _let((y = sqr(x_)) => go(xs.concat(y), y, n_ - 1));

  return go([], x, n);
};
```
Please note that `_let` has no type, i.e. you cannot express it in Typescript. However, we can easily type its invocations by explicitly specifiying the type of each default parameter (type assertion). Usually we want to avoid functions without a proper type but there seems to be no alternatives to obtain a syntax even remotely terse as with `_let`'s given implementation - at least none that I can think of. By the way, we will learn how to type all sorts of combinators in Typescript and delve into type theory in later chapters of this course.

### When the call stack vanishes

In some scenarios we cannot rely on the normal, synchronous function call stack but must use alternative structures.

#### Tail recursive functions

Tail recursion leads to eliminination of the function call stack. We will examine this omptimization technique in a later chapter. The implicit call stack is replaced with an explicit data structure that acccumulates the results of each recursive step. Hence it is called accumulator and holds the state of the recursive algorithm:

```javascript
const sum = xs => {
  const go = (acc, i) => // accumulator
    i === xs.length
      ? acc
      : go(acc + xs[i], i + 1);

  return go(0, 0);
};

sum([1, 2, 3, 4, 5]); // 15
```
[run code](https://repl.it/repls/ButteryMeanModels)

#### Asynchronous functions

Asynchronous functions cannot use the synchronous call stack, because at the time they are invoked all synchronous computations are already completed. We need an approach that somehow creates its own call stack when the asynchronous computations is carried out:

```javascript
const compCont = f => g => x => k =>
  g(x) (y => f(y) (k));

const sqrCont = x => k => setTimeout(k, 0, `sqrCont(${x})`);
const incCont = x => k => setTimeout(k, 0, `incCont(${x})`);
const log = x => console.log(`log(${x})`);

const main = compCont(sqrCont) (incCont) (2);

main(log); // log(sqrCont(incCont(2))) (A)
```
[run code](https://repl.it/repls/UtterDarkBytecode)

This is advanced functional programming so do not be intimidated. The decisive part is that line (A) evaluates to a nested function call tree that implicitly forms its own function call stack as soon as it is evaluated. This is where the asynchronous state is hidden.

If we formalize this further and add a couple of combinators we are going to wind up with the continuation type (`Cont`) and its associated monad. I will deal with this advanced functional programming idioms in another chapter of this course.

### Mimicking imperative state

The functional way of managing state may be hard to digest for imperative programmers. Let us alleviate the pain by mimicking imperative state with functions:

```javascript
const compState = f => g => h => x => s => {
  const [x_, s_] = h(x) (s);
    [x__, s__] = g(x_) (s_);
  
  return f(x__) (s__);
};

const sqr = x => x * x;

const addState = x => s => [x + s, s];
const modState = f => x => s => [x, f(s)];
const mulState = x => s => [x * s, s];

const main = compState(addState) (modState(sqr)) (mulState) (2);

main(3); // [15, 9] A
```
[run code](https://repl.it/repls/BruisedAgonizingMatrix)

This is again an advanced functional idiom but the underlying idea is simple: Instead of functions that just return a value we work with functions that additionally return the state. In the given example we pass the value `3` as the initial state to our main computation (A). In the first step the given state is multiplied with `2`, which yields a new return value. Then the state itself is modified by multiplying it with itself. At last both products are added. This yields the following expression `3 * 2 + 3 * 3`, which evaluates to `15` as the result value and `9` as the current state. Since we work with "stateful" functions both values the result and the current state are returned in a pair tuple like array.

If we keep abstracting and add a couple of combinators we will end up with the `State` type and its associated monad. I will deal with them in another chapter of this course.

[&lt; prev chapter](https://github.com/kongware/scriptum/blob/master/ch-1.md) | [TOC](https://github.com/kongware/scriptum#functional-programming-course-toc) | [next chapter &gt;](https://github.com/kongware/scriptum/blob/master/ch-3.md)
