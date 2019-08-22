# Introducing abilities

[langref]: languagereference.html

This tutorial explains how Unison handles effectful computations, like storing state or performing I/O, using **abilities**.  It assumes you haven't come across abilities before, and covers everything from the ground up.  If instead you just want a quick overview of the relevant language features and syntax, take a look at the [language reference][langref#abilities-and-ability-handlers].    

> Other languages with ability systems typically call them 'effect handlers' or 'algebraic effects', but many of the ideas are the same.

## Contents

- [Introducing abilities](#introducing-abilities)
  - [Contents](#contents)
  - [Motivation](#motivation)
    - [Making effectful behaviors visible in type signatures](#making-effectful-behaviors-visible-in-type-signatures)
    - [Decoupling interface from implementation](#decoupling-interface-from-implementation)
    - [Customizable control flow](#customizable-control-flow)
    - [Recent research](#recent-research)
  - [Using abilities](#using-abilities)
    - [Type signatures of functions using abilities](#type-signatures-of-functions-using-abilities)
      - [Type signatures in ability declarations](#type-signatures-in-ability-declarations)
    - [Examples of abilities](#examples-of-abilities)
      - [`Store`](#`store`)
      - [`IO`](#`io`)
      - [`Log`](#`log`)
      - [`Abort` and `Exception`](#`abort`-and-`exception`)
      - [`Choice`](#`choice`)
    - [More on abilities in type signatures](#more-on-abilities-in-type-signatures)
      - [Pure functions vs inferred abilities](#pure-functions-vs-inferred-abilities)
      - [Ability lists can appear before each function argument](#ability-lists-can-appear-before-each-function-argument)
      - [Ability inference and generalization](#ability-inference-and-generalization)
        - [Higher-order functions and ability polymorphism](#higher-order-functions-and-ability-polymorphism)
      - [Abilities are only relevant in computation signatures](#abilities-are-only-relevant-in-computation-signatures)
      - [Ability subtyping](#ability-subtyping)
      - [Defining functions with different ability lists on different arguments](#defining-functions-with-different-ability-lists-on-different-arguments)
  - [Invoking handlers](#invoking-handlers)
    - [Our first `handle` expression](#our-first-`handle`-expression)
    - [Trying out a test handler](#trying-out-a-test-handler)
    - [Stacking handlers](#stacking-handlers)
  - [Writing handlers](#writing-handlers)
    - [Example handlers](#example-handlers)
      - [Feeding in information via a pure handler](#feeding-in-information-via-a-pure-handler)
      - [Handling `Log`](#handling-`log`)
      - [Handling `Store`](#handling-`store`)
      - [Handling `Abort`](#handling-`abort`)
      - [Handling `Choice`](#handling-`choice`)
    - [The proxy handler pattern](#the-proxy-handler-pattern)
  - [Conclusion](#conclusion)
      - [What next?](#what-next)
  - [Exercises](#exercises)

## Motivation

Why does Unison have abilities, when so many languages get by without them?

### Making effectful behaviors visible in type signatures

One important reason is that when our functions have behaviors like storing state or doing disk I/O, we want to make that visible in their type signatures.  Here's an example of why this is useful.  

Suppose we're faced with this code:
``` haskell
f x = log (g x)
      2 * (g x)
```
`g` is some numerical function which we want to record in our log, and then double.  What if we find that `g` takes a lot of time to compute?  We'd want to refactor our code to compute it only once:
``` haskell
f x = v = g x
      log v
      2 * v   
```
Now, aside from the fact that it runs faster, is the new code equivalent to the old code?  Sure!  As long as `g` didn't have any **effectful** behaviours.  If it made a request to a network server, or opened a file on disk, then the code is not equivalent - it produces different network or filesystem traffic, and depending on external factors (like what the server returns), might yield a different return value.  

So, in this example, it's important to know whether `g` is effectful.  In Unison, we can tell by its type: we might end up with a type something like `Nat ->{Network} Nat` (we'll look harder at that syntax soon).  That's super useful!  We can see at a glance what effects the function might have, without having to read its code (and all the code it calls.)

### Decoupling interface from implementation

An 'ability declaration' presents an abstract interface representing the effectful primitives we'd like access to.  For example, the declaration of a `Network` ability might include `sendPacket` or `addPortListener`.  These **requests** are *not* bound directly to a specific implementation - they don't just directly call into the OS's sockets API.  Instead, we do that binding later, when we specify a **handler** for the ability.  Any specialization to OS sockets happens within the handler.  That gives us the flexibility to instead use a testing-specific handler, say one that records the packets sent and allows us to verify their contents.  Or we could take our 'live' handler, and stack it together with another handler that intercepts the packet contents and writes them to a log.  

Most languages have facilities to allow separation of interface from implementation.  But applying this principle to effectful APIs turns out to be a powerful way of structuring programs for modularity and testability.   

### Customizable control flow

Ability requests can be handled in such a way that they abort the computation.  This lets us write abilities like `Exception`, which provides the same kind of exception-handling facility as is found in other languages.  An example of other possible variations is an ability that helps with back-tracking solution search.  Putting control flow in the hands of programmers opens up plenty of possibilities for powerful library design, for example around concurrency and parallelism, without depending on case-by-case language features.  

### Recent research

The design of Unison's ability system comes out of recent research in the theory of programming languages - beginning in the 2000s.  In programming language terms this counts as cutting edge stuff!  It's only recently that we've learned how to build programming languages that combine the advantages listed above, in a user-friendly fashion.  See the links in the [language reference][langref#abilities-and-ability-handlers] if you're interested in further reading about the theory.  

## Using abilities

OK, let's get started.  Here's the declaration of an ability called `SystemTime`, which will let's us write code that can read the system clock.

``` haskell
ability SystemTime where
  -- Number of seconds since the start of 1970.
  systemTime : .base.Nat
```

It defines one **request**, `systemTime`, which returns the clock reading.  We'll come back to the term 'request' later.  

Let's use this to write some code.

``` haskell
tomorrow = '(SystemTime.systemTime + 24 * 60 * 60)
```

The only non-obvious thing here is the delay, that is, the use of the `'`.  

> 🐘 Remember that while `42` is a `Nat`, `'42` is a `() -> Nat` - that is, a function which takes an argument of the unit type, and returns a `Nat`.  See [delayed computations][langref#delayed-computations] for more detail.

We're using the `'` to turn `tomorrow` into a function rather than just a value.  Even though values are in a sense just functions that take zero arguments, as far as abilities are concerned they are different creatures.  Here's the key point: 

👉 Ability requests exist in the context of a function processing one of its arguments.

It's in the process of a function doing some computation that it makes sense to make a request using an ability.  A computed value can't do it - it's just sitting there, with no more computation to do.  

So the following wouldn't make sense:

``` haskell
-- wrong
tomorrow = SystemTime.systemTime + 24 * 60 * 60
```

On the one hand, this code would be saying we just want `tomorrow` to be a computed value, just a `Nat` we've got in the bag.  But on the other it's saying we want to compute it with the help of the system clock.  When do we want that computation to happen - when we first add it to the codebase?  That wouldn't make sense.  

So we need use add the `'` delay, to turn `tomorrow` into function, so we can trigger the computation at the right moment (and with the help of a handler, which we'll come to later). 

> Unison is a 'purely functional' language.  That means that evaluation of terms *cannot* in itself be effectful.  Adding the `'` delay is a consequence of that.  Any effectful code needs to pull its punches - it's not causing effects to happen during evaluation, but rather it's evaluating to a delayed function which can then explain to a handler what effects it would *like* to perform.  We'll see how that handler in turn can only either (a) turn the requested effects into pure computations, ready for evaluation, or (b) pass the buck, by translating them to requests in another ability.  If there are abilities like network or disk I/O, then the buck gets passed all the way to the very outside of the program, and into the Unison runtime system, using the `IO` ability.  
> 
> We'll find out more about the '`IO` on the outside' pattern when we meet the [`IO` ability](#io) later.

### Type signatures of functions using abilities

If we add `tomorrow` to the codebase, Unison tells us it's inferred the following signature:

``` haskell
tomorrow : '{SystemTime} .base.Nat
```
> 🐘 Again, that `'` is syntactic sugar for delayed function types - this signature is equivalent to `() ->{SystemTime} .base.Nat`. 

> 🐞 You may see a `∀` in the signature, due to Unison issue [#689](https://github.com/unisonweb/unison/issues/689).

The `{SystemTime}` is an **ability list** - in this case a list of just one ability.  It's saying that `tomorrow` _requires_ the `SystemTime` ability - that ability needs to be available in functions that call `tomorrow`.  And it's also saying that the `SystemTime` ability is available for use within the definition of `tomorrow` itself.  If a function of type `'.base.Nat` tried to make a `SystemTime.systemTime` request, Unison would reject it with an 'ability check failure': the ability required for that request is not in the set of *ambient abilities* (which is empty in this case).  

Suppose you're writing a function `foo` which should call `tomorrow`.  There are two ways of making the `SystemTime` ability available:
1. Put an ability list containing `SystemTime` in the signature of `foo`, the same as with the signature of `tomorrow`.  Indeed, if you leave the signature of `foo` unspecified, this ability list will be inferred again.  In this way the `SystemTime` requirement propagates up the function call graph.  
2. Use a handler.  This is how we stop that propagation.  We'll get on to handlers later.  

So now we've seen another key point about abilities:

👉 The requirement for an ability propagates from one function's signature to all its callers' signatures (until terminated by a handler).

This propagation is supported through the ability type inference mechanism as we've seen.  It's also _enforced_ by the type-checker - you can't get away with omitting one of these abilities if it's in fact required in your function (or a function it calls).  The typechecker makes sure that our ability lists faithfully reflect what's going on.  

#### Type signatures in ability declarations

Let's revisit the ability declaration we started with.

``` haskell
ability SystemTime where
  systemTime : .base.Nat
```

There's a significant piece of information that's been elided here for brevity.  The full and unabridged version of this declaration would be the following.

``` haskell
ability SystemTime where
  systemTime : {SystemTime} .base.Nat
```

Note the ability list that's appeared in the request signature.  Just as `tomorrow` has this ability in its signature, which therefore propagates to up to `foo` (in the example from the previous section), so `systemTime` did even beforehand, and it was this which propagated to `tomorrow` in the first place.  

The reason we can omit the name of the ability being defined from its requests' signatures is that, logically, it always needs to be there - using an ability's request requires that ability.  So as far as Unison is concerned, this bit of an ability's type signature 'goes without saying'.  You can write it either way.  

> 🤔 An ability declaration is the only place you'll see an ability list in a signature that doesn't follow the rule that 'ability requests exist in the context of a function processing one of its arguments' - i.e. the only place you can have a `{SystemTime}` not following a `->` or a `'`.

### Examples of abilities

#### `Store`

The following ability lets us write programs that have access to mutable state.

``` haskell
ability Store v where
  get : v
  put : v -> ()
```

💡 Notice that this ability has a type parameter, `v`.  Abilities can have these, just like type declarations can.  

The `Store` ability can be implemented using handlers, even though Unison does not offer mutable state as a language primitive - we'll see the implementation later.  

Here's an example, using `Store` to help label a binary tree with numerical indices, in left-to-right ascending order.

``` haskell
type Tree a = Branch (Tree a) a (Tree a) | Leaf

labelTree : Tree a ->{Store .base.Nat} Tree (a, .base.Nat)
labelTree t =
  use Tree Branch Leaf
  case t of
    Branch l v r ->
      use .base.Nat +
      l' = labelTree l
      n = Store.get
      Store.put (n + 1)
      r' = labelTree r
      Branch l' (v, n) r'
    Leaf -> Leaf
```

💡 Observe how the first branch of the `case` statement includes four side-effecting statements - the two lines with recursive calls to `labelTree`, and the lines in between.  Unison supports these **blocks** of statements, and handles the statements in sequence, because order of execution is important when running side-effecting code.  Note that the last line is in this case a non-side-effecting expression - the value of the block is just the value of this final expression.

> Note, the `'` in the identifiers `l'` and `r'` here are just part of the names - nothing to do with delay syntax this time.

#### `IO`

The main reason for having abilities is to give our programs a way of having an effect on the world outside the program.  There is a special ability, called `IO` (for 'Input/Output'), which lets us do this.  It's built into the language and runtime, so it's not defined and implemented in the normal way, but we can take a look at its ability declaration.

``` haskell
.base.io> view IO

  ability IO where
    send_ :
      Socket
      -> base.Bytes
      ->{IO} base.Either Error base.()
    getLine_ :
      Handle ->{IO} base.Either Error base.Text
    openFile_ :
      FilePath
      -> Mode
      ->{IO} base.Either Error Handle
    throw : Error ->{IO} a
    fork_ :
      '{IO} a ->{IO} base.Either Error ThreadId
    systemTime_ : {IO} (base.Either Error EpochTime)

    -- ... and many more requests, omitted here for brevity.
```

The `IO` ability spans many different types of I/O - the snippet above shows sockets, files, exceptions, and threads, as well as the system clock.  

> Typically you access these requests via the helper functions in the `.base.io` namespace, e.g. `.base.IO.systemTime : '{IO} EpochTime`.

So, since all the ways in which we can interact with the world are captured in the `IO` ability, why do we ever need any other abilities?  There are several reasons.
1. We don't want to write all our code in terms of low-level concepts like files and threads.  We want higher-level abstractions, for example persistent distributed stores for typed data, and stream-based concurrency.  The low-level stuff is what we're used to from traditional programming environments, but we want to hide it behind powerful libraries, written in Unison, that expose better abstractions.  
2. We don't want `{IO}` to feature too often in the type signatures of the functions we write, because it doesn't tell us much.  Since `IO` contains so many different types of I/O, it leaves the behavior of our functions very unconstrained.  We want to use our type signatures to document and enforce the ability requirements of our functions in a more fine-grained way.  For instance, it's useful that we know, just by looking at its signature, that `tomorrow : '{SystemTime} .base.Nat` isn't going to write to file or open a socket.  If we instead had `tomorrow : '{IO} .base.Nat`, then we'd have no such guarantee, without going and inspecting the code.  
3. Some things can be expressed well using abilities, but *don't* require interaction with the outside world. `Store` is an example.  

This leads us to a common pattern: 

👉 Typically, one ability is implemented by building on top of another.  And often, when we get down to the bottom of the pile, we'll find `IO`.  

For example, the handler for our `SystemTime` ability is going to require the `IO` ability, and it's going to call `.base.io.systemTime`.

In terms of the architecture of our programs, this typically means that the top level entry points for our 'business logic' are annotated with all the fine-grained abilities our program can use, like this:

``` haskell
placeOrder : Order ->{Database, Log, TimeService, AuthService} OrderConfirmation
```

And then we have one or more functions to wrap that logic, invoking handlers to collapse the signature down to one using only `IO`, like this:

``` haskell
orderServer : ServerConfig ->{IO} ()
```

> ##### Executing a Unison program
> 
> Here's the help for `ucm`'s `execute` command.
> ```
> .> help execute
> 
>   `execute foo` evaluates the Unison expression `foo` of type `()` with access to the `IO` ability.
> ```
>
> This shows us that we *need* to collapse our functions down to something like `orderServer`, so Unison knows how to run them.  

#### `Log`

Here's an example of an ability to let us append text to a log - for example a log file kept on disk.

``` haskell
ability Log where
  log : .base.Text -> ()
```

You could imagine the handler decorating the text with timestamps and other useful contextual information.  

The `Log` ability is typical of the class of abilities which let the program emit a sequence of data or commands.  The information flow is purely *out* of the computation using the ability.  Contrast this with the `Store` and `IO` abilities, in which the flow of information is two-way, both in and out of the computation.  

Most abilities are concerned with emitting and/or receiving data from/to the program.  However, abilities can do more than that: they can affect the program's control flow in ways that a regular function can't, as shown in the next example.  

#### `Abort` and `Exception`

The `Abort` ability lets us write programs which can terminate early.  

``` haskell
ability Abort where
  abort : a
```

Here's `Abort` in action:

``` haskell
use .base

getName : Input ->{Abort} Text
getName i = name = if not (valid i)
                   then Abort.abort
                   else extract "name" i
            canonicalName name

handleInput : Input ->{Abort} ()
handleInput i = name = getName i
                handleRequest name i
```

Suppose we're running `handleInput`, and we hit the `not (valid i)` error case inside `getName`: then we call `Abort.abort` and exit immediately.  Execution resumes from after the first enclosing `Abort` handler.  So, in this case, we exit both `getName` and `handleInput` immediately, since there's no handler in between the two.

> Note that the `abort` request has polymorphic type, `abort : a`.  This means it can be used in any context, and still typecheck.  It doesn't actually need to be able to return an `a`, because computation is not going to continue after the call to `abort`.  In `getName`, `abort` is being used where a `Text` is required, so `a` is instantiated to `Text`.  

There's a variant of `Abort`, which lets you provide a value to describe what's happened - this is analogous to the exception handling provided in some other languages.  

``` haskell
ability Exception e where
  throw : e -> a
```

😎 The ability mechanism is sufficiently general and powerful that what might otherwise be a whole separate single-purpose language feature, exception handling, instead becomes a few lines of library code.  Isn't that cool??

#### `Choice`

Here's another example - shown here to demonstrate further the idea of an ability affecting control flow.  

``` haskell
ability Choice where
  choose : .base.Boolean
```

There's a handler for this ability (which we'll see later), which gives the program not just one Boolean value after a call to `choose`, but both.  It then tries continuing the program under *both* conditions.  Each successive call to `choose` is a fork in the tree of possibilities.  The handler collects all the results from all the possible execution paths.  

This trick can be neat for exhaustively exploring a space of possibilities, for example to optimize some decision.  

That's the end of our tour of interesting example abilities.  Now let's dive deeper into the ability lists that can appear in type signatures, and what they mean.  

### More on abilities in type signatures

#### Pure functions vs inferred abilities

You can use an empty ability list to declare a pure function - that is, one that doesn't require any abilities.  For example:

``` haskell
inverse : Matrix ->{} Matrix
```

The typechecker then enforces that `inverse` does not require any abilities.  

👉 Telling Unison a signature `A ->{} B` is different from telling it `A -> B`.

The former is how you input the type of a pure function.  When you write the latter, you're asking for the ability list to be inferred by the Unison type-checker.  

👉 On code that you write, the signature `A -> B` doesn't mean 'no abilities', but rather that Unison will determine the ability list itself.  

This is an important distinction, and easy to forget, because the signature `A -> B` doesn't contain any visual cues to think about abilities.  

We'll learn more about the ability inference mechanism shortly, in [ability inference and generalization](#ability-inference-and-generalization).

#### Ability lists can appear before each function argument

So far we've seen functions whose types include one ability list, like so:

``` haskell
orderServer : ServerConfig ->{IO} ()
```
But the following is also possible:

``` haskell
orderServer' : ServerConfig ->{Log} '{IO} ()
```

> 🐘 This type signature is equivalent to `ServerConfig ->{Log} () ->{IO} ()`.

`orderserver'` is a function which, when partially applied to its first (`ServerConfig`) argument, can produce log messages, before finally yielding a function of type `'{IO} ()`.  That second function can then be forced, i.e. applied to `()`, to actually carry out some `IO`.  

The first application requires (only) the `Log` ability, and the second requires (only) the `IO` ability.  This is a useful distinction.  In this case, it tells us we can set up an order server, involving inspecting some configuration, in a computation that does some logging but is otherwise pure.  Only the second stage might do unrestricted `IO`.  

See [Defining functions with different ability lists on different arguments](#Defining-functions-with-different-ability-lists-on-different-arguments) for how to _define_ a function like `orderServer'`.

It's useful to keep the following in mind when reading type signatures.  

👉 Every `->` and `'` has an ability list `{…}` logically attached to it, describing the abilities required for applying the function to the preceding argument.  

'Logically', because as we've seen, the `{…}` can be left unspecified when writing code, to ask Unison to infer what it should say.  This process is described in the next section.  

#### Ability inference and generalization

Unison can do two levels of type inference for you.  The first is to infer the complete signature of your definition.

``` haskell
retries = 3
-- inferred type: .base.Nat
```

The second is to infer ability lists, wherever you have left them unspecified.  

```haskell
use .base

incrementP : Nat -> Nat
incrementP x = io.printLine "incrementP"
               x + 1
-- inferred type: Nat ->{io.IO} Nat               
```

Unison can see, from the use of `io.printLine`, that `incrementP` requires the `IO` ability.  

🚧 It's arguably surprising that Unison may infer a concrete ability for a function for which you provided a `Nat -> Nat` signature.  In future Unison will emit a message to say that it's done this.  ([#717](https://github.com/unisonweb/unison/issues/717))

When you do `add incrementP`, Unison will report the actual inferred type, `Nat ->{io.IO} Nat`.

So what does a plain `->` or `'` mean, when you see it after doing an `add`?  In this context it *does* mean a pure function - it's equivalent to `->{}` or `'{}`.

👉 When you *give* Unison a plain `->` or `'` (with no `{…}`) you're asking it to infer some abilities.  When Unison gives *you* a plain `->` or `'`, it means `->{}` or `'{}`.

So in particular, this means that 
- if you type `->{}`, Unison can render it back to you as just `->`
- if you want Unison to enforce that the function you are writing is pure, then specify a signature for it that uses a `->{}` or a `'{}`.

🚧 This dual meaning of a plan `->` arrow ('infer' or 'pure' depending on context) is a bit confusing.  The pure case may get its own style of arrow notation in future, to address this - Unison issue [#738](https://github.com/unisonweb/unison/issues/738).

> 🐞 Note, Unison can currently sometimes fail to output its inferred abilities when you do `view` or `edit` (although it does correctly output them at the `ucm` command line when you typecheck your code or do `add`/`update`.)  This is due to Unison issue [#703](https://github.com/unisonweb/unison/issues/703).  However, it will re-run its inference again when you next add the code.

##### Higher-order functions and ability polymorphism

Here's how Unison infers the type of `.base.List.map`, a higher-order function:

``` haskell
base.List.map : (a -> b) -> [a] -> [b]
-- inferred type: (a ->{𝕖} b) -> [a] ->{𝕖} [b]
```

It's added ability lists including a type variable, `𝕖`, in a process called **ability generalization**.  This is saying that, whatever the required abilities of the input function, the overall invocation of `map` will have the same requirements.

So for example, `'(base.List.map base.io.printLine ["Hello", "world!"])` has type `'{IO} [()]` - it requires `IO`, because it calls `printLine` which requires `IO`.  

We say that `base.List.map` is **ability polymorphic**: even though the function itself is in a sense pure, it can be used in a side-effecting way, depending on the ability requirements of its argument.  

The generalization process can work in tandem with inferring concrete abilities - for example:

``` haskell
applyP f x = .base.io.printLine "applyP"
             f x
-- inferred type: (i ->{𝕖, .base.io.IO} o) -> i ->{𝕖, .base.io.IO} o
```

This is saying that `applyP` requires `IO`, combined with whatever other abilities (`𝕖`) are required by its first argument.  (The combination process is a set union, so if `𝕖` also includes `IO`, then `IO` still only appears once in the resulting type.)

#### Abilities are only relevant in computation signatures

Not all type signatures are sensitive to abilities.  For example:

``` haskell
use .base

nowIfPast : Nat ->{SystemTime} Nat
nowIfPast t = now : Nat
              now = SystemTime.systemTime
              if t < now then now else t
```

The outer signature, on the top-level binding for `nowIfPast`, is what we'd expect.  But the signature on the inner binding for `now` is surprising.  Why doesn't it have to be something like `'{SystemTime} Nat`?  After all, the definition of `now` uses the `SystemTime` ability.  

The answer is that functions and lambdas define _computations_, and it is computations that can involve abilities.  The body of `now` involves a computation, but that computation is happening in the context of the outer function binding (which is where the `SystemTime` ability is mentioned).  The type signature on `now` is just talking about the _value_ that results from the computation - a plain `Nat`.

So, the signatures where abilities are relevant are just those for functions and lambdas.  Let's see what that looks like.

``` haskell
-- doesn't compile
nowIfPast' : [Nat] ->{SystemTime} [Nat]
nowIfPast' ts = f : Nat ->{} Nat
                f = t -> if t < SystemTime.systemTime then SystemTime.systemTime else t
                List.map f ts
-- also note that we're checking the system clock up to (2 * List.size ts) times in this example!
```  

In `nowIfPast'`, we've defined an inner lambda, `f`.  But we've made a mistake: the computation inside `f` involves the `SystemTime` ability, but `f`'s signature claims that `f` is pure (the empty braces `{}`).  Unison only accepts this function once we've removed the `{}` (to get ability inference) or replaced it with `{SystemTime}`.  

👉 Note that it's the _innermost enclosing lambda_ that specifies the available abilities.

So just because the signature on the top-level binding for `nowIfPast'`mentions `SystemTime`, that's not enough for Unison to accept `f`.

#### Ability subtyping

There's one last gotcha to be aware of when interpreting abilities in signatures.  Let's take a look at a better (if still slightly verbose) version of `nowIfPast'`.

``` haskell
nowIfPast'' : [Nat] ->{SystemTime} [Nat]
nowIfPast'' ts = now : Nat
                 now = SystemTime.systemTime
                 f : Nat ->{} Nat
                 f = t -> if t < now then now else t
                 List.map f ts
-- this time we check the system clock exactly once - much better! (unless `ts` was empty...)
```

`f` is now pure, which is nice - even though it's captured the value `now` which was produced in an effectful computation.  

The gotcha is that Unison will accept other signatures for `now` and `f` than those given above.
- For `f`, for example, it will accept `f : Nat ->{SystemTime} Nat`, saying that `f` is _allowed_ to use `SystemTime` even though it doesn't.  
- For `now`, it will accept `now : {SystemTime} Nat`, since (in the underlying type theory) `{SystemTime} Nat` is a subtype of `Nat`.  (🐞 This unhelpful permissiveness is Unison issue [#665](https://github.com/unisonweb/unison/issues/665).)

#### Defining functions with different ability lists on different arguments

In an [earlier section][#Ability-lists-can-appear-before-each-function-argument], we saw the following function signature:

``` haskell
orderServer' : ServerConfig ->{Log} '{.base.IO} ()
```
This sort of signature can be useful, to control exactly _when_ different effects take place.  

But we didn't see how to define such a function!  Here's a first, unsuccessful attempt.

``` haskell
use .base.IO

-- doesn't compile
orderServer' : ServerConfig ->{Log} '{IO} ()
-- remember this signature is equivalent to ServerConfig ->{Log} () -> {IO} ()
orderServer' sc unit = 
  log (ServerConfig.toText sc)
  startServer sc
  -- so this supposes we have a function startServer : ServerConfig ->{IO} ()
```

The problem with this is that by the time we've given `orderServer'` that `unit` argument, we've got on to the second arrow - the one that only allows us the `IO` ability.  So we can't use `log` in the function definition.  (If Unison allowed this, then partially applying `orderServer'` would yield a function of type `'{IO} ()` that uses the `Log` ability.)

🐞 Umm, actually at the moment the above does compile, wrongly ☹️  This is due to Unison issue [#745](https://github.com/unisonweb/unison/issues/745).

To define this function, we need to process one argument at a time, and at each stage only use the abilities that argument's arrow (the one on its right) gives us.  Here's a correct definition:

``` haskell
orderServer'' : ServerConfig ->{Log} '{IO} ()
orderServer'' sc = 
  log (ServerConfig.toText sc)
  '(startServer sc)
```

Note how we're just consuming the first argument, doing some logging, and then returning a lambda of type `'{IO} ()`.  

## Invoking handlers

Now it's time to take a look at _handle expressions_.  These are the things that actually let us run code that uses abilities.  

They do this by allowing us to _eliminate_ an ability from a type signature, including by converting it to another ability, which might be `IO`.  

### Our first `handle` expression

First let's see what the type signature of a handler looks like.  We'll see some implementations in [Writing handlers][#writing-handlers].  

```haskell
systemTimeToIO : .base.Request SystemTime a ->{.base.io.IO} a
```

First observation: we can treat a handler just like a regular function.  The only magic in its signature is the fact that `.base.Request` is a special built-in type.  We'll unpack the signature more later, but for now we can just read it as a function that takes `SystemTime` requests, and handles them using `IO`.  

Now let's remember our simple function from earlier, which uses `SystemTime`.

``` haskell
tomorrow : '{SystemTime} .base.Nat
tomorrow = '(SystemTime.systemTime + 24 * 60 * 60)
```

And now suppose we want to print the result to the console.  How can we do that?  Here's a good start:

``` haskell
use .base
use .base.io

printTomorrow : '{IO, SystemTime} ()
printTomorrow = '(printLine (Nat.toText !tomorrow))
```

Notice we've already introduced some `IO`, before even eliminating `SystemTime`, to print the result to the console.  

We can't run `printTomorrow` yet.

```
.> execute !printTomorrow

  The expression in red needs these abilities: {.base.io.IO, SystemTime}, but this location only has access to the {.base.io.IO} ability,
  
      1 | main_ = !printTomorrow
```

`execute` can only help us eliminate `IO` - any other ability we need to `handle` ourselves.  Here's how...

``` haskell
printTomorrow' : '{IO} ()
printTomorrow' = '(handle systemTimeToIO in !printTomorrow)
```

That works! 😎

```
.> execute !printTomorrow'
5638144744800
```

Notice in the function definition, we needed to force `printTomorrow`, turning it into `{IO, SystemTime} ()`, then delay the result again to get a `'{IO} ()`.  The intuition here is that you need to make `printTomorrow` actually _do_ its stuff, in order to handle the `SystemTime` requests it throws out - but that you need to delay the result because you can't have `printTomorrow'` doing `IO` requests except under a delay.  

❔ Would it be better for `handle` to take a delayed function, so you could just write `printTomorrow' = handle systemTimeToIO in printTomorrow`?  Feedback welcome... 📨

So now we can use a handler to execute code that uses abilities!  

There's something slightly unsatisfying about our definition of `printTomorrow` - its signature `'{IO, SystemTime} ()` tells us it needs _both_ abilities, but we wanted to _swap_ out `SystemTime` and replace it with `IO`.  We can smarten it up a bit by splicing the definition of `printTomorrow` into `printTomorrow'`.

``` haskell
printTomorrow'' : '{IO} ()
printTomorrow'' = '(handle systemTimeToIO in printLine (Nat.toText !tomorrow))

-- or, we can also do:

printTomorrow'' : '{IO} ()
printTomorrow'' = '(printLine (Nat.toText (handle systemTimeToIO in !tomorrow)))
```

### Trying out a test handler

We started this tutorial by singing the praises of handlers, and how the decouple the ability interface from a specific implementation.  Let's see that in action, and use a test handler to test our `tomorrow` function.  

Suppose we have a handler like this:

``` haskell
use .base
systemTimeToPure : [Nat] -> Request SystemTime a -> a
```

We can feed this handler a list of values we'd like it to return to each successive `systemTime` request.  And the resultant handled expression is pure - it doesn't need to map through to `.base.io.systemTime`.  

Let's test `tomorrow`:

``` haskell
use test.v1

test> tests.square.ex1 = run (expect ((handle (systemTimeToPure [0]) in !tomorrow) == 86400))
```

Awesome! 💥 😀

### Stacking handlers

Suppose our `labelTree` function from earlier reported its progress to a log.

``` haskell
use .base
labelTree : Tree a ->{Store Nat, Log} Tree (a, Nat)
```

And suppose we have the following.

``` haskell
-- Handlers for each ability.
logHandler : [Text] -> Request Log a -> (a, [Text])
storeHandler : v -> Request (Store v) a -> a
-- The first arguments to each of these are the initial values of 
-- the log and the store, respectively.  

-- Some data to work on.
tree : Tree Text
tree = Branch Leaf "Hi!" Leaf

-- A helper function.
fst : (a, b) -> a
```

Then here's how we run `labelTree`, eliminating both abilities.  We just need two nested handle expressions.

``` haskell
labelledTree : Tree (Text, Nat)
labelledTree = fst (handle logHandler [] in (handle storeHandler 0 in (labelTree tree)))
-- The call to fst just discards the log output.
```

Note that we could equally well have swapped the order we handle the two abilities.  

## Writing handlers

We now know how to use and handle abilities.  The last piece of the puzzle is writing our own handlers.  

Here's an example.  We'll unpack this piece by piece.  

``` haskell
use .base
use .base.io

systemTimeToIO : Request SystemTime a ->{IO} a
systemTimeToIO r =
  case r of
    { SystemTime.systemTime -> k } -> handle systemTimeToIO in k !systemTime
    { a } -> a

-- We've switched here to having SystemTime.systemTime return a 
-- .base.io.EpochTime, to match what .base.io.systemTime returns 
-- and so make things convenient.  
```

What's going on here?  Let's start with a recap of the type signature.  

* A `Request A T` is a value representing a computation that requires ability `A` and has return type `T`.  `.base.Request` is a built-in type constructor that ties in to Unison's `handle` mechanism: `handle h in k` is taking the computation `k` which has type `T` and uses ability `A`, building a `Request A T` for it, and passing that to `h`.

* A handler is a function `Request A T -> R` - it takes the computation and boils it down into some return value.  Common cases:

  * Typically the signature is `Request A t -> R`, where `t` is a type variable - i.e. `forall t. Request A t -> R`.  Usually the handler doesn't care what the computation's return type is, and we want to be able to apply it at any type.  

  * Sometimes the handler takes extra arguments - for example we saw `storeHandler : v -> Request (Store v) a -> a`, where the first argument is the initial value of the store.  The `Request` is always the last argument, so we can partially apply the handler to get a function whose type is of the above form, which a `handle` expression will accept.

  * Often, the result type `R` of the handled computation is the same as `T`, i.e. `Request A T -> T`.  We saw an exception with `logHandler : [Text] -> Request Log a -> (a, [Text])` - we wanted a way to get at the log that was written!

Let's pause and introduce the key idea behind handlers.  This is a bit subtle so take a deep breath!

👉 A handler handles _one step_ in a computation, and then specifies what to do with the rest of the steps.

We think of the computation as divided up into steps, punctuated by requests using the ability in question, and culminating in a final step which returns a result.  The handler specifies how to deal with each possible kind of step: each of the requests declared by the ability, plus the final step to get the return value.  

Once a handler has dealt with one step, it then specifies what to do with the **continuation** of the computation - that is, all the rest of the steps, treated as one unit.  Typically it does this by recursively calling itself, via `handle`.  So each invocation of the handler unwraps one step of the computation and deals with it.  

Make sense? 🤔  It should do once we've worked through some more examples.  

Let's start by unpacking the body of `systemTimeToIO`.  

* It's doing a pattern match on the `Request SystemTime a`, but the patterns are using a special syntax - here we're seeing `{ SystemTime.systemTime -> k }`.  This means, 'inspect the first step in the computation, and if it's a call out to `SystemTime.systemTime`, then match this branch, and take the remainder of the computation and call it `k`'.  `k` has type `EpochTime -> a`.  The argument of `k` is the return type of the ability request - i.e. the return type of `SystemTime.systemTime`.  This makes sense: if the ability is being used to retrieve some information, then probably the rest of the computation wants to use that information!  (If the request method had returned `()`, then `k` would have had type `() -> a` - there's no information being passed in, but still `k` is delayed.)

* It's handling this case first by evaluating `k !systemTime`.  This is the sharp end of the handler.  The point of the whole business was to consult the system clock by mapping down to the appropriate `IO` call.  That's what we do here.  Note the `!` to force that call, and how we pass the result straight into `k`, as input for the rest of the computation.  This part uses the `IO` ability, hence the `->{IO}` in the signature of the handler.  

* But we're not done yet!  We've only handled one step of the computation.  `k !systemTime` is all of the rest of the steps.  The handler _does_ specify how to handle all these: recursively, with the `handle systemTimeToIO in`.  That converts our effectful computation of return type `T` (in this case type `a`), into a computation of type `R` (in this case also type `a`), which does _not_ use the `SystemTime` ability.  

* The last case of the pattern match, ('the pure case'), `{ a } -> a` is easy.  It's saying 'and when we get to the last step of the computation, take the `a` it returns, and just pass that back directly as the result of the `handle` expression.'

Once we've covered all the cases, we've explained to Unison how to handle all the steps of a `SystemTime` computation, by translating it into an `IO` computation, in which the use of the `SystemTime` ability has been eliminated.  

And that's it!  So now let's take a look at some more examples.  

### Example handlers

#### Feeding in information via a pure handler

Let's revisit the `systemTimeToPure` handler which we were using for testing back in [Trying out a test handler](#trying-out-a-test-handler).  Here's the implementation:

``` haskell
use .base
systemTimeToPure : [Nat] -> Request SystemTime a -> a
systemTimeToPure xs r = case r of
  { SystemTime.systemTime -> k } -> case xs of
    x +: rest -> handle (systemTimeToPure rest) in k x
  { a } -> a
```

The interesting thing here is that the handler is taking a `[Nat]` argument, to give it a list of values to feed out each time there's a call to `systemTime`.  Note how the handler is partially applied in the handle expression, `(systemTimeToPure rest)`.  

> Note how the case statement fails to handle receiving an empty list `xs`.  Perhaps a better choice would be for the handler to use `Abort` to cover this case - see the [exercises](#exercises).

#### Handling `Log`

Now let's see a handler for our ability, `ability Log where log : Text -> ()`, which we met back [here](#log).  This one doesn't try to map it down to file I/O - it's just collecting the log lines and returning them in the handle expression's return value.

``` haskell
use .base
logHandler : [Text] -> Request Log a -> (a, [Text])
logHandler ts r =
  case r of
    { Log.log t -> k } -> handle logHandler (t +: ts) in !k
    { a } -> (a, ts)
```

Again, `logHandler` is being partially applied in the `handle` expression, for the same reason as with `systemTimeToPure`.  The _new_ things we see in this handler are...
* the return type - it's not just `a`.  The only `case` branch where that's visible is the one for the pure case.
* the `Log.log` request takes an argument.  That works how you'd expect - you just match on it (the `t` in the pattern.)
* the `!k`: because `log` just returns `()`, we only need to pass `()` when we call the continuation.  `!k` is another way of writing `k ()`.  

#### Handling `Store`

Remember the `Store` ability from [here](#store):

``` haskell
ability Store v where
  get : v
  put : v -> ()
```

Here's the handler for it.

``` haskell
use .base
storeHandler : v -> Request (Store v) a -> a
storeHandler storedValue s = case s of 
  { Store.get -> k } -> handle storeHandler storedValue in k storedValue
  { Store.put v -> k } -> handle storeHandler v in !k
  { a } -> a
```

This is all made up of pieces we've seen before.  Notice the trick in the `put` case: instead of passing through the old `storedValue` to the recursive call, we're passing the new value `v`.

It's good to dwell for a moment on where the stored state actually 'lives'.  It lives in the arguments passed on each call to the handler, and nowhere else.  The evolution of the state during the computation is captured by the changing successive values passed on each new recursive call.  

Now let's take a look at a couple of handlers for abilities that affect the program's control flow in ways that a regular function can't - the key here is whether and when they choose to call `k`.

#### Handling `Abort`

Here's the handler for `ability Abort where abort : a`, which we met in [`Abort` and `Exception`](#abort-and-exception):

``` haskell
use .base
abortToPure : Request Abort a -> Optional a
abortToPure r = case r of
  { Abort.abort -> k } -> None
  { a } -> Some a
```

The key point here is that the case for `abort` *does not use `k`*.  Whatever the remainder of the computation after the call to `abort`, it won't be evaluated, because the continuation `k` is discarded.  

#### Handling `Choice`

So if that was a handler calling the continuation 0 times, what about 2 times?  Let's see a handler for `ability Choice where choose : Boolean`, which we met [here](#choice).  This handler runs through a whole tree of possible evolutions of the computation, with a fork at each `choose`, and collects the results in a list.

``` haskell
use .base
choiceToPure : Request Choice a -> [a]
choiceToPure r = case r of 
  { Choice.choose -> k } -> (handle choiceToPure in k false) ++ (handle choiceToPure in k true)
  { a } -> [a]
```

> This is the first handler we've seen where the call to `handle` is not in tail position - i.e. where the return value of `handle` still needs some further processing (with `++`) before returning.  Recursive calls in tail position can be made any number of times in sequence, while still using constant space (because the function's stack frame can be reused from call to call).  `choiceToPure` does not have this property.  In this case that's probably fine - if you're handling a computation that makes a long sequence of calls to `choose`, you're likely to run into the exponential growth of the `[a]` list before the linear growth of the handler stack troubles you.  (See the [exercises](#exercises) to try writing a handler that does random sampling instead of accumulating all possible results.)

### The proxy handler pattern

Sometimes, you want to handle an ability without eliminating it - so, passing through the requests, perhaps with some modifications, and perhaps 'teeing off' some information as it goes through.

For example, suppose we want a handler that logs the values that a `Store` computation `put`s.

``` haskell
use .base
storeProxyLog : (v -> Text) -> Request (Store v) a ->{Store v, Log} a
storeProxyLog print r = case r of
  { Store.get -> k } -> 
    v = Store.get
    handle storeProxyLog print in k v
  { Store.put v -> k } -> 
    Log.log ("Put: " ++ print v)
    Store.put v
    handle storeProxyLog print in !k
  { a } -> a
```

This `Store` handler is itself using `Store`!  And `Log` too.  

Take a look at the stack of handlers we end up with in `result` below.

``` haskell
computation : '{Store} Nat
computation = 'let
                Store.put 42
                Store.get

result : (Nat, [Text])
result = handle logHandler [] in (handle storeHandler 0 in (handle storeProxyLog Nat.toText in !s))

> result

    9  | > result
           ⧩
           (42, ["Put: 42"])

```

So the abilities used by the computation are being transformed step-by-step as follows: `{Store Nat}`, then `{Store Nat, Log}`, then `{Log}`, then `{}`.  

> Note we can't just say `handle storeProxyLog print in k Store.get` in the `get` branch: then the `Store.get` would end up being handled by `storeProxyLog`, instead of passed out to the next `Store` handler.

Note how, if we wanted it to, `storeProxyLog` would be able to *change* the `v` value that `storeHandler` stores on a `put`, and the value that gets given to the computation on a `get`.  

> ⚙️ Interestingly, you can define your own handler for `IO`!  One application for this would be to write a proxy handler to record all the input a program receives from the outside world.  You could then write another handler to replay the information into the program later - which is a pure computation, and so easier for debugging.  (🍭 This would be a great pair of Unison library functions for someone to write! 😀)

## Conclusion

You now know all about Unison abilities!  You can...
* write effectful code that calls out to abilities like `Log`, `Abort`, and `Store`
* run effectful computations using `handle` expressions
* write handlers (functions of type `Request r a -> o`) to translate one ability into another (possibly `IO`), including handlers that
  * send information *in* to the computation, or receive information coming *out*, or both
  * influence program control flow in ways regular functions can't (like `Abort` and `Choice`)
* understand ability lists in subtle type signatures like `ServerConfig ->{Log} '{IO} ()`
* understand Unison's ability generalization and inference, and when it lets you
  * omit ability lists in type signatures, letting Unison infer them
  * pass effectful functions into higher-order functions like `map`
* structure a program into a core part which uses fine-grained abilities, and then transform that using outer layers of handlers to yield a program of type `'{IO} ()`.

Great work! 👍 😎 

#### What next?
* Try working through some of the exercises below!
* We'd love you to get in touch with us to let us know what you found difficult, or if you have any ideas for improving Unison's ability support, or even if you just think it's cool 😁

## Exercises

Here are some ideas for exercises to get you fluent in working with abilities.  In each case, be sure to actually try running your code!

1. Write an ability `ConsoleIO`, and a handler for it of type `Request ConsoleIO a ->{IO} a` that uses `getLine` and `putLine` from `.base.io`.  Write a program of type `'{ConsoleIO} Text`.  

2. Write a handler for `Log` that writes to file.  Write a program of type `'{Log} ()` and try running it first with your handler, and then with the `logHandler` from [Handling `Log`](#handling-log).

3. Write a handler for the `Exception` ability shown in [`Abort` and `Exception`](#abort-and-exception).

4. Change the `systemTimeToPure` handler from [Feeding in information via a pure handler](#feeding-in-information-via-a-pure-handler) so that it uses `Abort` to handle the case where it runs out of data to pass to the computation.  

5. Change the `choiceToPure` handler from [Handling `Choice`](#handling-choice) to return a value of `type Choices a = Choice (Choices a) (Choices a) | Result a`, so you can see what choices led to each result.  Try using it in a program that counts 'tails' in a sequence of coin tosses.  

6. Write an ability `Random` that lets a program ask for a random `Nat`.  Write a handler that acts as a pseudo-random number generator.  (You get a simple PRNG by iterating the function `seed -> (6364136223846793005 * seed + 1442695040888963407)`) [[reference](https://en.wikipedia.org/wiki/Linear_congruential_generator)]

7. Write a handler for `Choice` which instead of enumerating *all* possible variants of the computation instead takes a random sample of them.  

8. Start with `ability Send a where send : a -> ()` and `ability Receive a where receive : a`.  Write a 'multi-handler' of type `Request (Send m) a -> Request (Receive m) b ->{Abort} b`.  Then enhance it so that the messages `m` flowing between the two computations are output using `Log`.  TODO is this possible?  does it need more coverage in this tutorial?

9. Write a proxy handler for `IO` that records the input received by the program, at least for some `IO` requests.  Write another `IO` handler to play it back into the program later.  Write a function `recordReplay : ('{IO} ()) ->{IO} ()` that runs its argument twice - once 'for real' with recording, and once as a replay.  (Don't bother saving off the record to disk - that will get easier when Unison gets support for typed data persistence.)
