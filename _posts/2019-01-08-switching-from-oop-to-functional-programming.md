# Switching from OOP to Functional Programming

Why functional programming is so hard?

![](https://cdn-images-1.medium.com/max/2000/1*W8fpVDNFk2Rr_xuShYATJw.jpeg)

> Q: I’ve heard a lot of good things about functional programming but I find it
> very difficult to understand. I have years of experience in
C++/Java/C#/Javascript/etc but it doesn’t help, it feels like learning to code
from scratch again. Where should I start?

Switching to FP style indeed requires a mindset change. You no longer have your
usual primitives, like classes, mutable variables, loops, etc. You will not be
productive for the first couple of months, you will be stuck for hours or days
on some simple things that usually took minutes. It will be hard and you will
feel stupid. We all did. But after it clicks you will gain superpowers. I don’t
know a single person who switched back from FP to OOP after doing FP daily. You
may switch to a language that doesn’t have FP support but you’ll still be
writing using FP concepts, there are that good.

It this article I’ll try to break down some concepts and answer common questions
that bugged me when I was learning FP.

1.  [There are no
classes](https://medium.com/@olxc/switching-from-oop-to-functional-programming-4187698d4d3#7cbe)
1.  [All you need is a
function](https://medium.com/@olxc/switching-from-oop-to-functional-programming-4187698d4d3#8215)
1.  [No, you can’t change a
variable](https://medium.com/@olxc/switching-from-oop-to-functional-programming-4187698d4d3#c99a)
1.  [No, you can’t do ‘for’
loops](https://medium.com/@olxc/switching-from-oop-to-functional-programming-4187698d4d3#ab33)
1.  [Your code is not a list of instructions
anymore](https://medium.com/@olxc/switching-from-oop-to-functional-programming-4187698d4d3#8795)
1.  [On
](https://medium.com/@olxc/switching-from-oop-to-functional-programming-4187698d4d3#8866)`null`[s
and
exceptions](https://medium.com/@olxc/switching-from-oop-to-functional-programming-4187698d4d3#8866)
1.  [Functors, Monads,
Applicatives?](https://medium.com/@olxc/switching-from-oop-to-functional-programming-4187698d4d3#fca2)

### 1. There are no classes

> Q: No classes? How do I structure my code then?

Turns out you don’t need classes. Like in a good old procedural programming your
program is just a collection of functions except in FP those functions have to
have some properties ([discussed
later](https://medium.com/@olxc/switching-from-oop-to-functional-programming-4187698d4d3#33c3))
and they also have to *compose. *You will hear the word ‘composition’ a lot as
this is one of the core ideas of FP.

I would suggest to stop thinking about ‘creating instances of a class’ or
‘calling class methods’. Your program will be just a bunch of functions that can
call each other.

<span class="figcaption_hack">Slide from
[https://slidle.com/jivko/functional-programming](https://slidle.com/jivko/functional-programming)</span>

Side note: a lot of FP languages have a notion of a ‘type class’ which shouldn’t
be confused with OOP understanding of a class. Type classes purpose is to
provide *polymorphism. *You don’t have to worry about it too much at first but
if you’re interested check out this article: [Type classes
explained](https://medium.com/p/a9767f64ed2c?source=your_stories_page---------------------------).

> Q: What about data? I often have some data and functions that change that data
> in one class.

For that we have Algebraic Data Types (ADT), which is just a fancy name for a
record that holds data.

<span class="figcaption_hack">Scala example</span>

<span class="figcaption_hack">Haskell example</span>

You can think about it as a class that only has a constructor and nothing else.
Using FP terminology they are ‘Types’ and the constructors are called ‘Type
constructors’. This is how you construct types and get values out of them:

<span class="figcaption_hack">scala</span>

<span class="figcaption_hack">haskell</span>

Note that in Haskell `name` and `age` are actually functions that take a value
of type `Person` and return its fields.

> Q: Ok, but how do I change the person’s age, for example?

Changing things in place (in imperative programming understanding) is a mutation
and you can not do mutation in FP ([more on that
later](https://medium.com/@olxc/switching-from-oop-to-functional-programming-4187698d4d3#c99a)).
If you want to change something — you make a copy of it.

<span class="figcaption_hack">scala</span>

<span class="figcaption_hack">haskell</span>

There are 2 kinds of ADT worth knowing: **product** type and **sum** type.

* Product type: a collection of fields, all have to be specified in order to
construct a type:

* Sum type: represents optionality. Either your type is something **or** something
else. Example, a *Shape* can be a *Circle* **or** a *Square*.

<span class="figcaption_hack">scala</span>

ADTs can be nested as well: a *Shape* is a sum type where each option can be a
sum or a product on its own. Any kind of domain model can be represented as a
combination of sum and product types.

> Q: Why sums and products are so special?

Besides being basic building blocks for modeling they are natively supported by
most of FP languages. Product types can be deconstructed and statically checked
while sum types can be used for pattern matching:

<span class="figcaption_hack">scala</span>

<span class="figcaption_hack">haskell</span>

### 2. All you need is a function

Meet your new best friend — a function. You may know it by different names:
getter, setter, constructor, method, builder, static function, etc. In OOP those
names are associated with different contexts and have different properties. In
FP a function is always just a function — it takes values as input and returns
values as output.

There is no need to instantiate anything to use functions (as there is no
classes), you just import the module where the function is defined and just call
it. A functional program is just a collection of ADTs and functions, as in
`Shape`s example above.

There are 3 main properties a function should have:

1.  **Pure**: no side effects. Functions are not allowed to do more than their type
definition says. For example, a function that takes an `Int` and returns an
`Int` cannot change global variables, access filesystem, do network requests,
etc. It can *only* do some transformations on the input and return some value.
1.  **Total**: returns values for all inputs. Functions that crash or throw
exceptions on some inputs are not total or *partial*. For example a `div`
function: type declaration promises that it takes an `Int` and returns an `Int`
however if the second argument is `0` it will throw ‘division by zero’
exception, hence it’s not total.
1.  **Deterministic: **returns the same result for the same input. For deterministic
function it doesn’t matter how and when it’s called — it will always return the
same value. Functions that depend on a current date, clock, timezone or some
external state are not deterministic.

Most programming languages cannot enforce these properties statically so its
programmer responsibility to satisfy those properties. For example, Scala
compiler will happily accept functions that impure, partial and non
deterministic:

<span class="figcaption_hack">scala</span>

In Haskell, on the other hand, you can’t (easily) write a function that isn’t
pure or non deterministic: any kind of side effecting function will return an
`IO` which is a value that *represents* ‘side effectful’ computation. Totality
property is still on the programmer, as you can throw exceptions or return so
called bottom which will terminate the program.

<span class="figcaption_hack">haskell</span>

> Q: Why do I care if a function has these properties or not?

If a function satisfy those properties you get “referential transparency” ([more
detailed in a separate
article](https://medium.com/@olxc/referential-transparency-93352c2dd713)). In
short, you’ll get the ability to look at the function type definition and know
exactly what it can and cannot do. You can refactor your code fearlessly because
RT guarantees you nothing will break. RT is basically what allows us to control
complexity of our software. Refactoring in OOP can be a nightmare as you don’t
know which objects call what and when until you actually run the program and
build a mental model in your head. And even then it’s not an easy task.

### 3. No, you can’t change a variable

> Q: This is the weirdest part, how do I make anything useful without changing
> variables?

If you have a variable `person` that is bound to `Person("Bob", 42)` you cannot
reassign it to `Person("Bob",43)`. What you *can* do is to create a different
variable by creating a copy and specifying what you want to be changed (as we
discussed before). Variables are immutable and used only to alias or **label
values**, not as a physical reference or a pointer to the actual data.

> Q: Why not just change it in place?

Because it breaks referential transparency and, as I said before, referential
transparency is the key to FP. It will make your life so much easier while not
having mutable variables is a fair price to pay. Besides, no mutation means you
get thread safe code for free, no more weekends wasted on ‘happens only on a
Tuesday evening’ concurrency bugs.

Immutability is a simple concept but it’s hard to adopt after years of OOP
experience. It’s common to see people reverting to `var`s in Scala just ‘to get
this thing working’. It’s fine to do that at first but always look for an
immutable implementation. Besides, there is no such ‘hack’ in Haskell so you
have to be immutable from day 1.

### 4. No, you can’t do ‘for’ loops

> Q: Our bread and butter — the ‘for’ loop — you say FP doesn’t have it as well?
> How do you iterate over an array?

No mutation meaning no ‘for’ loops, as it usually mutates some counter ‘i’ until
some predicate is met. However, we have other means of achieving the same —
recursion and higher order functions.

#### Recursion

You have to get comfortable with recursion as it is everywhere in FP. For
example, a sum of all numbers in a list will look like this:

<span class="figcaption_hack">scala</span>

<span class="figcaption_hack">haskell</span>

It’s common to work with recursive data structures, like lists or trees. Even
natural numbers can be expressed [in this
way](https://wiki.haskell.org/Peano_numbers). Natural way of traversing those
structures is by pattern matching on type constructors and apply recursive
functions to the recursive parts of the data structure. A general pattern is to
first define a base case, such as an empty list case to terminate recursion and
then define a general case.

#### Higher order functions

Higher order functions take other functions as an argument. Talking about
iterations you have to know how to use `map` and `fold`:

<span class="figcaption_hack">scala</span>

<span class="figcaption_hack">haskell</span>

> Q: What’s up with names? `map`? Isn’t like `foreach`?

Yes, but only for lists. Soon you will found out that `map` is not about
transforming a list but a has a different semantics depending on what we want to
*map*. If you want to know more — lookup `Functor`, which is a higher kinded
type that provides a mapping interface. But don’t worry about `Functor`s too
much—just think of `map` as a function that knows how to iterate over data
structures, such as lists, trees, dictionaries, etc.

`fold` also has a deeper meaning and relates to `Foldable`. The intuition is
that it takes some data structure and produces a single value, such as sum. Note
that `map`, unlike `fold`, applies function to each value independently while
`fold` can carry some sort of accumulator that depends on previous values.

There are much more functions but knowing those 2 can get you a long way for
most iteration problems.

### 5. Your code is not a list of instructions anymore

In imperative language you could do this:

These functions have ‘side-effects’, e.g. they **do something**. The result of
their actions is a changed state of the entire program — some files have been
written to the disk, output in the console, updated internal entities map, etc.
Once you call such function — it’s done, completed, executed.

> Well, nothing new here, this is how I usually program.

Sure, but in functional program **nothing** is executed until the very last
moment. Your functions have to take values and return values, no side-effecting
allowed. The output of one function is an input for some other function which,
in its turn, creates an input for some other function and so on.

This is how that program will look like in FP:

Note the `unsafeRun` function (let’s say its provided by the language). Before
`unsafeRun` all we’ve done is gluing functions together, nothing is executed. We
are building some sort of execution plan — “this function has to be called
first, then based on it’s output we will call one of those two functions” and so
on.

It is also not an easy concept to grasp, as we used to throwing some additional
behavior here in there that **does** things, like logging statements or sets
some flag, clears a queue, etc. You no longer can get away with that as these
additional functions have to follow the types and compose with other functions.
And this is a good thing — it forces us to be more principled about what our
program is doing and make sure that everything is encoded within the function’s
type signature.

### 6. On `nulls` and exceptions

Nulls are all over imperatively written code bases. The problem with `null` is
that it’s a lower level abstraction leaked into higher level type system. If I
see a function that returns a `Person` then (if a function is total) I expect to
get a `Person` that has a name, address, whatever. The `null` is not a person.
`null` is often used to represent absence or some sort of internal failure that
prevents function from returning a proper value. If a function can somehow fail
to return a `Person` it should say so in its *type definition*. In FP we can
represent absence with a sum type:

<span class="figcaption_hack">scala</span>

<span class="figcaption_hack">haskell</span>

If a function returns a `Maybe` or an`Option` of `Person` it explicitly says —
`Person` is not guaranteed. The caller will *have* to check if the returned
value is `Some` or `None`, that means no more `null` dereferencing problems or
`null` pointer exceptions.

If you think about it, `null` is kind of a low level primitive that relates to
the runtime system rather to your program logic. When you write in a higher
level languages with garbage collection you don’t really care when and how the
objects are allocated in memory, nor what is the generated machine code for your
function is. This is what higher level languages are for — they create an
abstraction so you don’t have to think about the details. `null` breaks this
abstraction so the code becomes polluted with weird `p != null` checks or even
worse — dereferencing problems.

Similarly, exceptions. There is no need for a special mechanism with a special
syntax just to deal with exceptional cases. In your pure program it’s possible
to represent absence, failures and exceptions with ordinary *values*. Throwing
exceptions with `throw e` makes function *partial* (non total) which again
breaks referential transparency and creates problems.

If you work with JVM and use java libraries you will *have* to deal with
exceptions. And it’s ok to use exception is some special cases, like `IO`, but
make sure it’s part of a function type — a caller has to know that function
throws, what kind of exceptions can be thrown and those promises can be checked
at compile time.

### 7. Functors, Monads, Applicatives?

> Q: I hear FP people talk about this things constantly but they don’t make any
> sense to me. Is there an easy explanation?

People have discovered general patterns and gave them names from category
theory. Functors, Monads and Traversables are pretty powerful and common
abstractions, you will see them everywhere. It’s probably a topic for an article
on its own. But for now— don’t worry about it. You will learn about them
eventually (or maybe even re-invent them yourself). Get comfortable with
function composition, higher order functions and polymorphic functions. Then
read about [type
classes](https://medium.com/@olxc/type-classes-explained-a9767f64ed2c). After
that Functors and Monads should come naturally. The takeaway here is that there
is no magic and there isn’t much more to it than we have already discussed in
this article — pure functions and function composition.

*****

Hope it was helpful and if not — please send me your feedback. As someone said,
“once you understand Monads you loose the ability to explain it to others”, so I
hope this article wasn’t too far from what OOP developers usually experience.
Thanks for reading and enjoy your FP journey.
