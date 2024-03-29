# Kleisli composition

In this article I would like to introduce you to two common composition operators
and explain some of their utility, using first a made up example, and then an
example from the real world. These operators are all about composing functions
of slightly weird types. For while regular function composition combines functions
of types `a -> b` and `b -> c`, these guys concern themselves with combining
functions of types which don't naturally fit perfectly together.

The Kleisli arrow is often written using the fishbone operator.
`f >=> g` is the Kleisli composition of the functions `f` and `g` with the following abstract types.

```elm
f: a -> M b
g: b -> M c
f >=> g : a -> M c
```

If we consider lists of numbers as a concrete example, the types would be

```elm
f: Num -> List Num
g: Num -> List Num
```

and their composition would also have the type `Int -> List Int`.
Such a function might for instance be the square root function, such as

```elm
4 -> [2, -2]
9 -> [3, -3]
...
```

If you're using complex numbers you can also consider the n'th root function,
where it will produce a list of `n` roots.

The function `doubleSquareRoot` which takes the square root of a number twice
can be defined as the Kleisli composition of this square root function with itself.

```elm
doubleSquareRoot = sqrt >=> sqrt
```

To understand what `doubleSquareRoot 16` will evaluate to, we will first consider
the first `sqrt` evaluation.
`sqrt 16` will produce `[4, -4]`.
For each of the results in this list, the next `sqrt`
function will be evaluated:

```elm
sqrt 4 == [2, -2]
sqrt -4 == [2i, -2i]
```

These results are then consolidated to produce the final result.

```elm
doubleSquareRoot 16 == [2, -2, 2i, -2i]
```

This technique can be used to chain together many computations to
produce a larger result.
When thinking about lists this can be understood as modeling problems
with many results such as finding roots, or maybe finding the possible
next moves in a graph traversal problem.

## The Bind operator

The bind operator is often written `>>=`.
Instead of composing two functions like the Kleisli arrow, the bind
operator combines a value `x` and a function `f`.
As with the arrow, we can think about the types of `x >>= f`.

```elm
x: M a
f: a -> M b
x >>= f : M b
```

As an example, consider once again the square root function.
If we would like to compute the square roots of several values,
such as the numbers 4, 9 and 16, we might write

```elm
[4, 9, 16] >>= sqrt
```

which would evaluate to `[2, -2, 3, -3, 4, -4]`.

The Kleisli arrow and bind operator are very similar.
One way to think about it is that they _mean_ the same thing,
but which one you use is a matter of practicality and the shape of
the data you happen to have.

## A real world example

At work we have recently been experimenting with using F# for a simple
web api.
An endpoint might look like

```F#
let updateEvent id =
    getBody<Models.WriteModel>
    >> Result.map models.writeToDomain
    >> Result.bind (Service.updateEvent id)
    >> Result.map commitTransaction
    >> Result.map models.domainToView
```

This constructs a larger function using the regular `>>` function composition operator.
The input for this function will be a `context` object which contains the body of the
HTTP request as well as a reference to the database, which the `Service` uses.

Let's break this down line by line.

- **Line 1** `getBody` is a function which takes the context object and produces a `Result.Ok WriteModel` if it can, or a `Result.Error` if it can't.
- **Line 2** `writeToDomain` is a function which takes a `WriteModel` and produces a `DomainModel`. In our case, this cannot fail, hence the need for `Result.map`. If it could fail, we would instead use `Result.bind`.
- **Line 3** `Service.updateEvent id` is a function which takes a `DomainModel` and produces a `DomainModel`. In addition it needs a reference to the context object to produce the side effect we want, namely updating the database. This can also fail, which is why we need `Result.bind`.
- **Line 4** `commitTransaction` takes any input it is give, let's call it `'a`, and the context object, and after commiting the transaction to the database, it will return its input `'a`. The reason it also returns its input is to pass it along the pipeline of functions.
- **Line 5** `domainToView` takes the `DomainModel` and produces a `ViewModel`.

We use `Result.map` to handle errors along the way using the built-in `Result` type.
For instance `getBody` might fail if the incomming JSON cannot be parsed as a `Models.WriteModel` type.
The `Result.bind` function reveals that the `Service.updateEvent` function itself
may fail and produce a `Result.Error`.

This seems like it will work, until you consider which functions needs use of the context object.
`getBody` obviously needs the context to access the HTTP body.
In addition, both `Service.updateEvent` and `commitTransaction` needs a reference to the database.
The way we solve this is to consider `Service.updateEvent id` not as a function from the domain
model to `Result<DomainModel>` but as a function from the domain model to `ctx -> Result<DomainModel>`.
That is, a function which takes domain models and produce domain models and can fail but also needs a
context.
That might sound and indeed be confusing, but if you don't think too much about it, it kind of makes sense.
Using Kleisli composition's closely related cousin the bind operator `>>=`, we can rewrite our function.[0]

```F#
let updateEvent id =
    getBody<Models.WriteModel>
    >> Result.map (models.writeToDomain id)
    >>= Result.bind (Service.updateEvent id)
    >>= Result.map commitTransaction
    >> Result.map models.domainToView
```

It's a small change, but now our two functions can access the context.
The way I think about it, is that this is regular composition of functions,
but when you need the context object, you inject it using the bind operator instead of regular composition.
And that is all you need to know to use this pattern productively!

## Technicalities

If you're interested, the bind operator may be defined as follows.

```F#
f >>= g = fun ctx -> g (f ctx) ctx
```

The reason we are using the bind operator here instead of the Kleisli fishbone operator,
is that the very first function _only_ takes the context as a parameter.
Had the first function also taken a parameter in addition to the context,
you would just replace all instaces of `>>=` with `>=>`.
The implementation of the Kleisli arrow might then look like

```F#
f >=> g =
    fun x ->
        fun ctx ->
            g (f x ctx) ctx
```

It might be difficult to see the similarities between functions that produce lists of
things and functions that produce functions which need contexts, but they are there.
They have some similar properties and can be understood in the same way. And of course,
if you don't need to, you don't have to think too much about it to be able to use
composition operators such as the bind and Kleisli arrows effectively.

[0]: It is well known that [monads don't compose](https://blog.tmorris.net/posts/monads-do-not-compose/),
and both `Result` and functions which need contexts are monads.
This is only a problem if you're trying to write code that works generically with
two different types of abstract monads, and this is where people use monad transformers.
However, we only need these specific, concrete monads, which allow us
to write a single `>>=` bind operator which handles all of our needs.
