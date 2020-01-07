---
title: "Trampolining and stack safety in Scala"
classes: wide
categories:
  - scala
tags:
  - functional programming
  - scala
---

## Making every call a self recursive tail call

![image-center]({{ site.url }}{{ site.baseurl }}/assets/images/posts/trampoline/header.png){: .align-center}

## Problem

Scala compiler isn't great with tail call elimination in general. This makes functions composed of many smaller functions prone to stack overflows.

```scala
def even[A](lst: List[A]): Boolean = {
  lst match {
    case Nil => true
    case x :: xs => odd(xs)
  }
}

def odd[A](lst: List[A]): Boolean = {
  lst match {
    case Nil => false
    case x :: xs => even(xs)
  }
}

even((0 to 1000000).toList) // blows the stack
```

## Solution

The Scala compiler is able to optimize a specific kind of tail call known as a self-recursive call:

```scala
def gcd(a: Int, b: Int): Int =
  if (b == 0) a else gcd(b, a % b)
```

We will try to find a way to transform every call into a self-recursive call that compiler will optimize.

## Thoughts

As in example above, multiple functions can be invoked in a recursive process but we can have only one self-recursive function to make it stack safe. We need to change our functions to build a description of a program instead of actually making any recursive calls. Then, we would need some sort of generic function that will go through our description and evaluate each recursion step.

## Approach #1

Let’s start with algebraic data type that would represent our recursive program:

```scala
sealed trait Trampoline[A]
  
case class Done[A](value: A) extends Trampoline[A]
  
case class More[A](call: () => Trampoline[A]) extends Trampoline[A]
```

**Trampoline** is the term for this technique, think about it as a _program description_.
The program description can be either:

* `Done` if there are no computations to be done and we can yield a value
* `More` if there is a recursive function call to be made

Here’s an example of a simple program that has 2 suspended computations and yields value 42

```scala
More(() => More(() => Done(42)))
```

Of course we won’t create it manually, our recursive functions will:

```scala
def even[A](lst: Seq[A]): Trampoline[Boolean] = {
  lst match {
    case Nil => Done(true)
    case x :: xs => More(() => odd(xs))
  }
}

def odd[A](lst: Seq[A]): Trampoline[Boolean] = {
  lst match {
    case Nil => Done(false)
    case x :: xs => More(() => even(xs))
  }
}
```

So now when we call `even(0 to 1000000)` we get a value with type `More[Boolean]` containing a function to compute `odd` for a list length `999999`. To get the actual boolean value we need to traverse to the bottom of all nested `More`s to finally reach `Done`. Let’s write a function for that:

```scala
def run[A](trampoline: Trampoline[A]): A = {
  trampoline match {
    case Done(v) => v
    case More(t) => run(t()) // <- tail recursive, yay
  }
}
```

`Done` case is straightforward, `More` case has to first execute our suspended function to get either `More[A]` or `Done[A]` and recursively call run on that. This kind if recursion is safe and will be optimized by the compiler.

One small refactoring that looks unnecessary but it will be useful a bit later — separate executing a single step from executing all steps:

```scala
def resume[A](t: Trampoline[A]): Either[() => Trampoline[A], A] = t match {
  case Done(v) => Right(v)
  case More(k) => Left(k)
} 

def run[A](t: Trampoline[A]): A = resume(t) match {
  case Right(value) => value
  case Left(more) => run(more())
}
```

And also for convenience we can embed these functions into `Trampoline` trait:

```scala
sealed trait Trampoline[+A] {
  def resume: Either[() => Trampoline[A], A] = this match {
    case Done(v) => Right(v)
    case More(k) => Left(k)
  }
   
  final def runT: A = resume match {
    case Right(value) => value
    case Left(more) => more().runT
  }  
}
```

There you go, stack safe computations ftw.

## Still having stack overflows…

Let’s say you have a State monad implemented like this:

```scala
case class State[S, +A](runS: S => (S, A)) {
  def map[B](f: A => B): State[S, B] =
    State[S, B](s => {
      val (s1, a) = runS(s)
      (s1, f(a))
    })

  def flatMap[B](f: A => State[S, B]): State[S, B] =
    State[S, B](s => {
      val (s1, a) = runS(s)
      f(a) runS s1
    })
}
```

And a usage example:

```scala
def zipIndex[A](as: Seq[A]): State[Int, List[(Int, A)]] =
  as.foldLeft(pureState[Int, List[(Int,A)]](List()))((acc,a) => for {
    xs <- acc
    n <- getState
    _ <- setState(n + 1)
} yield (n, a) :: xs)

zipIndex(0 to 1000000).runS(0) // blows the stack
```

Looks innocent but what we don’t see is many `flatMap` calls chained together consuming all stack memory during execution.

```scala
flatMap(flatMap(flatMap(...
```

Different use case but the same problem: `flatMap` is not self-recursive tail call. Can we make it one? Let’s take `flatMap` implementation, do the same mechanical transformation (as we did for `even` and `odd` example) by changing return type to be `Done` or `More`:

```scala
case class State[S, +A](runS: S => Trampoline[(S, A)]) {
  def flatMap[B](f: A => State[S,B]): State[S, B] =
    State[S,B](s => More(() => {
      val (s1, a) = runS(s).runT // <- not in the tail position
      More(() => f(a) runS s1)
    }))
}
```

`runS` has to be executed one the state `s` _before_ returning `More` meaning that our trampolining trick would not work.

## Approach #2

Instead of trying to fix `flatMap`'s implementation we can just suspend the whole `flatMap` operation on the `State` and defer the binding until our `run` and `resume` functions will be called.

```scala
sealed trait Trampoline[A]

case class Done[A](value: A) extends Trampoline[A]

case class More[A](call: () => Trampoline[A]) extends Trampoline[A]

case class FlatMap[A, B](
  sub: Trampoline[A],
  cont: A => Trampoline[B]) extends Trampoline[B]
```

`FlatMap` is basically a signature of `flatMap` stored in a data structure. After that State's `map` and `flatMap` would look like this:

```scala
case class State[S, +A](runS: S => Trampoline[(S, A)]) {
  def map[B](f: A => B): State[S, B] = State[S, B](
    runS andThen { tramp => {
      val (s, a) = tramp.runT
      Done((s, f(a)))
    }}
  )

  def flatMap[B](f: A => State[S, B]): State[S, B] = State[S, B](
    runS andThen { tramp => {
      FlatMap[(S, A), (S, B)](tramp, { case (s, a) => f(a).runS(s) })
    }}
  )
}
```

This trick allowed us to change nested `flatMap(flatMap(flatMap...` calls to a data structure `FlatMap(FlatMap(FlatMap(...` trading stack for heap.

Now we should be able to compose our `“trampolined”` States and safely use "for" comprehensions as in `zipIndex` example. One thing left is to add newly added `FlatMap` case to our tail recursive interpreter function:

```scala
sealed trait Trampoline[+A] {
  def resume: Either[() => Trampoline[A], A] = this match {
    case Done(v) => Right(v)
    case More(k) => Left(k)
    case FlatMap(sub, cont) => sub match {
      case Done(v) => cont(v).resume
      case More(k) => Left(() => FlatMap(k(), cont))
      case FlatMap(sub2, cont2) =>
        (FlatMap(sub2, (x:Any) => FlatMap(cont2(x), cont)):Trampoline[A]).resume
    }
  }

  final def runT: A = resume match {
    case Right(value) => value
    case Left(more) => more().runT
  }
}
```

So the `FlatMap` having `sub` (sub expression) and `cont` (continuation function) is about matching on the `sub`:

* `Done(v)` means sub is the last in the chain of `FlatMaps` and we just have to thread `v` through `cont` and call `resume`
* `More(k)` (where `k` is a suspended function) means we can make a step by calling `k`, making the resulting trampoline a `sub` expression of a new `FlatMap` with the same continuation
* On `FlatMap(sub2, cont2)` we will create a `FlatMap` with the next `sub2` expression. Its continuation will create another `FlatMap` with evaluated inner `cont2`.

Also, we must cast explicitly to `Trampoline[A]` for the compiler to be able to figure out that this is in fact a tail-recursive self-call.

## Still having stack overflows...

Even with our improvements it is possible to blow a stack in cases when `FlatMap` nesting is too deep: evaluating a `sub` to make a step can cause another `sub` to be evaluated and so on. So let’s not directly create deeply nested `FlatMaps` (like in State's `flatMap`) but use a helper function that will do the following:

```scala
sealed trait Trampoline[+A] {
    
  /* resume and runT */
    
  def flatMap[B](f: A => Trampoline[B]): Trampoline[B] = this match {
    case x => FlatMap(x, f)
    case FlatMap(sub, k) => FlatMap(sub, (x: Any) => k(x).flatMap(f))
  }
}
```

Yes, this turns out to be `Trampoline`'s `flatMap`. If its called on either `Done` or `More` we will just wrap it in a `FlatMap`. If its called on the `FlatMap` then we will reassociate the bind to the right.

With this change in place let’s refactor `State`'s `flatMap` to use `Trampoline`'s `flatMap`:

```scala
case class State[S, +A](runS: S => Trampoline[(S, A)]) {
  def map[B](f: A => B): State[S, B] = State[S, B](
    runS andThen { tramp => {
      val (s, a) = tramp.runT
      Done((s, f(a)))
    }}
  )

  def flatMap[B](f: A => State[S, B]): State[S, B] = State[S, B](
    runS.andThen { (tramp: Trampoline[(S, A)]) => {
      tramp.flatMap { case (s, a) => f(a).runS(s) }
    }}
  )
}
```

Also, our interpreter function can also create deeply nested `FlatMaps` so we need to use `Trampoline`'s `flatMap` as well:

```scala
sealed trait Trampoline[+A] {
  
  def resume: Either[() => Trampoline[A], A] = this match {
    case Done(v) => Right(v)
    case More(k) => Left(k)
    case FlatMap(sub, cont) => sub match {
      case Done(v) => cont(v).resume
      case More(k) => Left(() => FlatMap(k(), cont))
      case FlatMap(sub2, cont2) =>
        sub2.flatMap((x:Any) => cont2(x).flatMap(cont)).resume // <-
    }
  }

  /* runT, flatMap */
  
}
```

## Putting it all together

We started with data structures `Done` and `More` to represent program description and used self-recursive interpreter function to avoid stack overflows. It was suitable for simple use cases but was insufficient for more complex cases, like nested monadic flatMaps. We represented a `flapMap` operation in a `FlatMap` data structure but faced the same problem: evaluation of deeply nested `FlatMaps` can still cause stack overflows. We solved that by delegating the creation of `FlatMap` to a Trampoline’s `flatMap`.

--- 

This article is based on Rúnar Bjarnason’s paper [“Stackless Scala With Free Monads”](http://blog.higher-order.com/assets/trampolines.pdf) that I definitely recommend checking out for more detailed explanation and see how Free monad is actually a generalization of a Trampoline monad.