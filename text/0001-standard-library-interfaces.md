- Feature Name: belt-monads
- Start Date: 2018-05-26
- Issue/PR: https://github.com/BuckleScript/bucklescript/pull/2843 

# Summary
[summary]: #summary

# Motivation
[motivation]: #motivation

As the BuckleScript/ReasonML ecosystem grows through library contributions,
which are evident by introduction of the Redex package registry, it's
impossible to overlook that we are lacking common abstractions when several
libraries are reimplementing them.


Some of these libraries are:
* `Belt.Option` from [Belt](https://github.com/BuckleScript/bucklescript/blob/master/jscomp/others/belt_Option.ml#L30-L46)
* `Belt.Result` from [Belt](https://github.com/BuckleScript/bucklescript/blob/master/jscomp/others/belt_Result.ml#L33-L49)
* `Future` from [reason-future](https://github.com/RationalJS/future/blob/master/src/Future.re#L28-L37)
* `Effect` and `Affect` from [bs-effects](https://github.com/Risto-Stevcev/bs-effects/blob/master/src/Effect.re#L18-L36)
* `Stream` from [bs-highland](https://gitlab.com/scull7/bs-highland/blob/master/src/Highland.re#L29)

The expected outcome of this RFC is to arrive at a collection of common
interfaces to be included in BuckleScript's standard library, Belt, to allow for
a consistent and coherent ecosystem of libraries to grow.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

> **Disclaimer**: this section explains how to use the proposed functors as if
> they did already existed in the library, and expects no knowledge of them or
> the concepts they embody.

Say we have an `option` type defined, allowing us to represent the _presence_ or
the _absence_ of a value.

```reason
type option('a) =
  | Some('a)
  | None;
```

A value such as `Some(1)`, of type `option(int)`, will represent the _presence_
of some number, such as 1. A value of type `None`, of type `option(int)` in this
case, will represent the _absence_ of a number.

In order to manipulate the contents of an `option('a)` value, we can use the
`switch` operator:

```reason
let add_1 = x =>
  switch(x) {
  | Some(y) => y + 1
  };
```

As we started defining the function above, everything looked great. Until the
compiler warned us that we were missing _branches_ in that switch. We missed a
the `None` constructor!

So we go back an add it:

```reason
let add_1 = x =>
  switch(x) {
  | Some(y) => y + 1   
  | None => ??? + 1
  }
```

But we realize that we have nothing to add 1 to! We could simply return `0` but
that would be misleading, since callind `maybe_add_1(Some(-1))` would also
return `0`.

Perhaps we want to simply add 1 if there is _some_ number, and if there isn't
one we just say that we didn't get any number.

```reason
let maybe_add_1 = x =>
  switch(x) {
  | Some(x) => Some(x + 1) /* if there is a number, add 1 to it */
  | None => None           /* if there isn't a number, simply return none */
  };
```

This idiom is quite common, and we use it quite frequently with types that are
used as _containers_. That is, types that simply hold some data in them.

Let's make it more reusable so we can reuse it for a function that will take 1
from a number: `substract_1`

```reason
let maybe_run_f_on_contents = (f, x) =>
  switch(x) {
  | Some(y) => Some(f(y))
  | None => None 
  };
```

Great! Now we can rewrite both those functions as:

```reason
let maybe_add_1 = maybe_run_f_on_contents( x => x + 1 );
let maybe_sub_1 = maybe_run_f_on_contents( x => x - 1 );
```

Much cleaner. 

The truth is that this pattern is so common that it even has a name. It's name
is `map`, and we have seen this function on Lists or Arrays (which are
containers for more than one value).

But what if we had that `map` function that works on our `option('a)` type? We
then would not need to go about `switch`ing or implementing it ourselves, which
we need to do for every new type, so easy to make mistakes! We're only human
after all.

If we open the `Belt.Interfaces` module, then we have access to _functors_
that help us build modules so that they are always type-safe, well-behaved, and
can be optimized by the compiler.

Let's try one of them, the `Monad.Make` functor.

```reason
open Belt.Interfaces

module Option = {
  type option('a) =
    | Some('a)
    | None;

  include Monad.Make(
      {
        type t('a) = option('a);

        let return = x => Some(x); /* teach the functor how to create a value
                                      of your type */

        let bind = (o, f) =>       /* and teach it what to do with a function */
          switch (o) {             /* that uses the current contents but will */
          | None => None           /* return another value of your type       */
          | Some(x) => f(x)
          };
      },
    ): Monad.S with type t('a) := option('a);
}; 

let maybe_add_1 = Option.map( x => x + 1 );
let maybe_sub_1 = Option.map( x => x - 1 );
```

Ahh, even better!

Now that this type we created works just like one from the standard library,we 
can share it with other people and they can expect it to be well-behaved and
soundly typed too! 

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The reference implementation code was proposed in [PR #2843: [Proposal] Belt
Monads](https://github.com/BuckleScript/bucklescript/pull/2843). What follows
is a refined proposal of the ideas and code included there.

### The Interfaces Module

We begin by defining a module of common interfaces, `Belt.Interfaces`, that will
re-export a set of _functors_ that allow for quickly and safely constructing
modules.

```reason
/* file belt_Interfaces.ml */ 

module Monad = Belt_Monad;
module Alternative = Belt_Alternative;
/* the way we include and re-export these modules is subject to change */
```

This will allow developers to quickly `open Belt.Interfaces` and have access to 
a collection of useful abstractions.

In order to optimize for developer experience without compromising on
type-safety, each module will define `module type` that will include the
absolute minimum necessary information needed to construct the full module, and
a `Make` functor that will take as an input that minimum module, and will create
an instance for the indicated type(s).

Sample module skeleton:

```reason
module type Base = {
  /* the absolutely minimum module required to create an instance of this interface */
  type t('a);
};

module type S = {
  /* the complete interface that will be created from the Base module */
  type t('a);
};

module Make = (M: Base): S => {
  /* the functor that will create an instance of S from a Base module */ 
  type t('a) = M.t('a);
};
```

### The Monad Interface and Functor

As part of this proposal, a set of very common functions including `map` and
`flatMap`, are presented under the name of `Monad`. Full signature below:

```reason
module type S = {
  /*
    Canonical names, which can certainly be dropped in favor of the user-friendly
    ones.
   */

  /* create a value of this type */
  let return: 'a => t('a);

  /* transform the contents and collapse it into a new value of this type */
  let bind: (t('a), 'a => t('b)) => t('b);

  /* collapse a value that is wrapped twice, such as Some(Some(1)) into Some(1) */
  let join: t(t('a)) => t('a);

  /*
     User-friendly named operations
   */

  /* create a value of this type: Option.of(1) */
  let of: 'a => t('a);

  /* apply ~f to the value in this type */
  let map: (t('a), ~f: 'a => 'b) => t('b);

  /* transform the contents and collapse it into a new value of this type */
  let flatMap: (t('a), ~f: 'a => t('b)) => t('b);
};
```

Briefly, a `Monad` is a _context_ where we can apply functions; the `option` type
we defined in the first section can behave as one, and so can `result`, `list`,
`array` (for convenience), `stream`, and `future`/`promise`.

This means that a big number of incredibly commonplace datatypes can benefit of
soundly-typed functions that are automatically built for them.

The initially proposed set of _functors_ includes one for types parameterised
over a single type variable, and one for types parameterised over two type
variables: `Monad.Make` and `Monad.Make2`. Both are necessary for the sound
construction of the types `option` and `result` using these functors.

For the `Monad.Make` functor the minimum module type could look like:

```reason
module type Base = {
  /* defines the type we will be working with */
  type t('a);

  /* defines how to deal with a value of such t, and a function that returns
     another value of t */
  let bind: (t('a), 'a => t('b)) => t('b);

  /* defines how to construct a value of t */
  let return: 'a => t('a);
};
```

For other functors it may look differently, but still only include the absolutely
necessary.

### The Alternative Interface and Functor

Representing alternatives between values is fairly common in programming.
Operators such as `||` (boolean or) are defined such that if the former element
is of a certain value, the latter is preferred. This can be leveraged greatly
to succintly build very complex logic, for example:

```reason
let is_admin = user => user.is_admin || user.id == 0 || user.name == "Admin";
```

In a similar fashion, if we needed to provide _fallbacks_ to certain _absent_ values
represented with the `option('a)` type, we need to do unpack them to deal with
with _absent_ case:

```reason
let do_search : string => option(user);

let default_user = { name: "Admin" };

let search = name => {
  let result = do_search(name);
  switch(result) {
  | Some(user) => user
  | None => do_other_search(name) |. Option.getWithDefault(default_user)
  }
};
```

Which lacks the ergonomics of a `<|>` operator or an `orElse` function that would allow for:

```reason
let search = name =>
  do_search(name)
  |. orElse(do_other_search)
  |. orElse(do_yet_another_search)
  |. orElse(Some(default_user))
```

The full signature of the `Alternative` module is:

```reason
module S = {
  /* the type we will be providing an alternative for */
  type t('a);

  /* the function that given two of these values will pick one */
  let orElse : t('a) => t('a) => t('a);
}
```

And an implementation for `Option` could be:

```reason
/* belt_Option */
open Belt.Interfaces;

include Alternative.Make({
  type t('a) = option('a);
  let orElse = (a, b) =>
    switch(a) {
      | Some(_) => a  /* if the first parameter has a value, return it */
      | None => b     /* if the first parameter has no value, return the second */
    };
});
```

This of course can also be implemented for any type.

### The Foldable Interface and Functor

Fairly often we find ourselves in situations where we want to extract a final
value of a computation, considering all the possible outcomes.

For a `list(int)` we may want to extract a single integer representing the sum
of all the elements in the list, with an empty list returning 0. Which we commonly
know in Javascript as `reduce` (albeit it behaves slightly differently).

For a `result(auth_level, error)` we may want to always extract an `auth_level`
even when there has been an error.

For an `option(user)` we may want to fallback to a predefined `guest` user when
there is no user.

In all of these cases, we want to consider all possible constructors available
for a given type, and come up with a single value, of the same type in all
cases.

Informally speaking, we want to _fold_ over a particular datatype, and this is a
very common and useful abstraction to have.

The full signature of the `Foldable` interface looks like:

```reason
module S = {
  /* the type we are expected to workw with */
  type t('a);

  /* the function that does the folding, it expects:
      * a value t('b) that will be folded
      * an initial value ~init
      * a function ~f that will decide what to do given t('b) and ~init */
  let fold_left:  (t('a), ~f='('a, 'b) => 'a, ~init='a) => 'a;  
};
```

More comprehensive versions of this interface (and perhaps more lawful ones)
can be provided, but might not be necessary.

An example usage of it for the type `result('a, 'b)` would be:

```reason
/* belt_Result.ml */
open Belt.Interfaces;

include Foldable.Make2({
  /* define the type we will be working with */
  type t('a, 'b) = result('a, 'b);

  /* define the behavior of `fold_left` */
  let fold_left = (value, ~f, ~init) =>
    switch(value) {
    | Ok(_) => f(value, init)   /* if we have some value, use f to compute the result */
    | Error(_) => init          /* if we have an error, return the init value instead */
    }
});
```

### Performance Considerations

After brief experimentation, it appears that the compiler will inline the functor
calls if they are happening in the same module. 

Using functors in separate files will lead to an additional function
application when first loading the modules themselves.

However, while this does incur in the cost of creating the module at _runtime_,
it should only occur _once per module_ and only _on load time of such module_.

While it does not look like it could be a problem, measuring the initialization
time could be of interest.

# Drawbacks
[drawbacks]: #drawbacks

If we don't pick a level of abstraction carefully, we may raise the entry bar
significantly for people that want to collaborate in Belt, and who want to
produce modules that are well-behaved and idiomatic.

This applies both to extensive use of categorical structures, such as splitting
interfaces hierarchichally (that is, `Monad.Make` being a composed functor that
will underneath use `Functor.Make` and `Applicative.Make`), and to exhaustive
collections Java-style interfaces (such as `Iterable`, `Iterator`, `Collection`,
`Collector`, etc).

# Rationale and alternatives
[alternatives]: #alternatives

- Let library authors manually and only optionally adhere to certain type
  signatures  for their work, allowing someone to reimplement `flatMap` in a
  completely unexpected and inconsistent way.
- Let library authors invoke a third party dependency to provide this
  interfaces at arbitrary levels of granularity, some perhaps extensively
  categorical, some others not.

# Prior art
[prior-art]: #prior-art

Interfaces in particular have been around for quite some time, and are a proven way of 
ensuring consistency, but also a great way of conveying _intent_.

Functional programming languages in particular have opted for categorical
structures such as Monads to provide sound, rigid, and powerful interfaces on
which to build common data types that make programming a more predictable task.

Languages such as Haskell, Idris, Scala, include in their standard libraries a
rich collection of interfaces modelled in a similar fashion as the ones
proposed here, with the difference that they are perhaps more involved in the
granularity they offer.

Libraries such as Javascript's RxJS, Fantasy-Land, or FolkTale have succeeded
in reusing these several of these abstractions to construct incredibly useful
tools for developers to leverage on a daily basis.

A lesson to be learned from the above is that going deep down the rabbit hole
(or rather high up the ivory tower) has in the past being tied to difficulty of
adoption. Thus, I'd like to see us make very pragmatic choices with little to
no compromise on type-safety.

# Unresolved questions
[unresolved]: #unresolved-questions

- What exactly are the interfaces that Belt should initially provide besides
  the ones proposed here?
- Can compiler optimizations be performed to always attempt to inline functor calls?
