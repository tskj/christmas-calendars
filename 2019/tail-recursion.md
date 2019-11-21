# Tail recursion

Tail recursion is a special way of writing recursive functions such that a compiler can optimize the recursion away and implement the algorithm as a loop instead.
This is not because loops are inherently faster, but because every function call generally incurs the cost of a new stack frame in which it executes, sometimes leading to the dreaded `stack overflow` - where the beloved site of the same name gets its name.
Let's investigate how this works.

But first we need to consider what recursion is and which functions are tail recursive and which are not.
A function is said to be recursive if it is defined in terms of itself.
As is customary when thinking about recursion, we will look at the factorial function.
In mathematics we might see a definition such as `n! = n * (n-1) * ... * 2 * 1`.
If the author cares to define the same function recursively you might see

```math
n! = n * (n-1)!    if n > 0
n! = 1             if n = 0
```

Notice how the definition is of two parts, a recursive part and a base-case part.
This translates nicely to programming.
In a functional language we might define the factorial function like the following.

```F#
let factorial n =
    match n with
    | 0 -> 1
    | n -> n * factorial (n-1)
```

The syntax is slightly different, but the structure is suprisingly similar.
This function is however not tail recursive, it is merely recursive.

A slightly re-written version, which _is_ tail recursive, might look like the follwing.

```F#
let factorial acc n =
    match n with
    | 0 -> acc
    | n -> factorial (acc * n) (n-1)
```

This definition is mostly the same, with the addition of a new variable `acc`, short for accumulator.
Introducing an accumulator is often necessary to be able to rewrite a function to be tail recursive.
Annoyingly, this version has to be called with an initial value of `acc=1`, so for instance `5!` would look like `factorial 1 5`.

The crucial difference between these definitions, however, is that the recursive call to `factorial` in this new definition is in "tail position".
In a sense, the last operation that is performed in the body of the function definition, is to call itself.
This is what allows the compiler to replace the function call with a loop instruction and reuse the same stack frame - because there is no more computation to be done in this stack frame after the recursive call.
When it returns, all we do is return that answer.
In which case we might as well return the answer directly.

Maybe this becomes more clear if we take a loot at the same two definitions re-written more verbosely, using intermediary variable names and if statements, instead of the more terse mathematics-like syntax above.

```reasonml
let factorial n =
    if (n == 0) {
        n
    }
    let rest = factorial (n-1)
    n * rest
```

```reasonml
let factorial acc n =
    if (n == 0) {
        acc
    }
    let sofar = acc * n
    let next = n - 1
    factorial sofar next
```

The important difference is that in the first definition there is more work to be done after the recursive call, namely the multiplication `n * rest`, while in the second definition all the work happens before.

One way I like to think about this in a mutable way is that the last line `factorial sofar next` is an instruction to go back to the top of the function and binding `acc` to this new value `sofar` and `n` to `next`.
Whenever `n == 0` and it is time to break out of this looping recursion, we simply return the latest value of `acc` directly to the original caller of the factorial function.
In fact, this is precisely how Clojure implements tail recursion.
Instead of detecting it automatically, Clojure forces you to use a special syntax with the special keywords `loop` and `recur`.
Here is a Clojure implementation.

```clojure
(defn factorial [n]
  (loop [acc 1
         n n]
    (if (= n 0)
      acc
      (recur (* acc n) (dec n)))))
```

Again we see the same structure, only expressed using a different syntax.
