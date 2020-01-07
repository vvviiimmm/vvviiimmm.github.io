---
title: "Free Monads Explained"
classes: wide
categories:
  - scala
tags:
  - functional programming
  - scala
---

## Building composable DSLs

![image-center]({{ site.url }}{{ site.baseurl }}/assets/images/posts/freemonad/header.jpeg){: .align-center}

## 🤔 Free monad?

> The core idea of Free is to switch from effectful functions (that might be impure) to plain data structures that represents our domain logic. Those data structures are composed in a structure that is free to interpretation and the exact implementation of our program can be decided later.
{: .notice--info}

## 💩 Imperative example
As a use case example we will take a program that asks for user’s name a greets him:

```scala
println("Greetings!")
println("What is your name?")
val name = scala.io.StdIn.readLine()
println(s"Hi $name!")
```

As usual, we will go through series of small transformations trying to generalize things and eventually come up with Free monad implementation.

## 📖 Creating an algebra
First off, lets define an algebra that represents our program. There are two clear and distinct operations:

1. `Tell` represents an action to tell something to a user. We don’t say how, it might be a standard out print, message on a screen, etc.
2. `Ask` questions user for something and returns the answer.

```scala
sealed trait UserInteraction[A]

case class Tell(statement: String) extends UserInteraction[Unit]

case class Ask(question: String) extends UserInteraction[Unit]
```

The imperative program from above can be described as a `List` of `Ask`s and `Tell`s:

```scala
val program = List(
  Tell("Greetings!"), 
  Ask("What is your name?"), 
  Tell("Hi, nice to meet you!"))
```

Instead of calling functions we construct a _description of a program_. In order to execute such program we need to know how to “execute” each individual `UserInteraction` and then simply map it over the list:

```scala
def execute[A](ui: UserInteraction[A]): A = ui match {
  case Tell(statement) =>
    println(statement)
  case Ask(question) =>
    println(question)
    val answer = scala.io.StdIn.readLine()
    () // ignoring the answer for now
}
 
def run[A](program: List[UserInteraction[A]]): Unit = program.foreach(execute)
```

The interpreter (`execute` + `run`) is an _implementation_ of our program description, this is the place where all mutation and side effects can happen.

By the way, our new program is slightly different from the first imperative program — instead of greeting user by his name it just says generic “nice to meet you”. Our `UserInteraction` data structure is not very useful right now, we don’t have a way of getting the values of previous `Ask`'s and referring them in `Tell`. The steps are sequential and depend on the value of a previous computation. Sounds like a `Monad`, right? If our `UserInteraction` would be a `Monad` (and lets even say we already implemented `pure` and `flatMap`) that would allow us to rewrite our program like this:

```scala
val program = for { 
  _    <- Tell("Greetings!")
  name <- Ask("What is your name?")
  _    <- Tell("Hi $name!")
} yield ()
```

Now we can compose our user actions and access computation results. The problem is that program not a `List` of instructions anymore:

```scala
def run[A](program: UserInteraction[A]): Unit = ??? // match on program?
```

By introducing monadic bind we lost the ability to introspect the data structure, there is no way of knowing what our resulting `UserAction[A]` was made of and interpret each step.

## 😴 Monadception
First, let’s change our `Ask` to ‘return’ a value of a proper type in order to capture it in the monadic bind:

```scala
sealed trait UserInteraction[A]

// An effect that takes a String and retuns Unit           vvvv
case class Tell(statement: String) extends UserInteraction[Unit]

// An effect that takes a String and retuns a String     vvvvvv
case class Ask(question: String) extends UserInteraction[String]
```

We know that `UserInteraction` needs to be monadic, be able to compose sequential computations but at the same time preserve information about the computation steps. The `flatMap` operation takes an `F[A]`, a function `A => F[B]` and returns an `F[B]`, this is where information is lost, same as with regular functions composition we lose information about what were the original composed functions. **What if we could go meta again and replace calls to `flatMap` with a data structure**? Eventually I want to get something like this:

```scala
FlatMap(Tell("Greetings!"), (_) => 
  FlatMap(Ask("What is your name?"), (name) => 
    Return(Tell(s"Hi $name!"))))
```

By introducing `FlatMap` and `Return` we captured what it’s like for something to be a `Monad` and this is what `FreeMonad` is:

Imagine the `F[_]` being our `UserInteraction` but it can be any type constructor and there are no constraints on the `F` being a `Monad` or something. `Return` and `FlatMap` are the analogy of `pure` and `flatMap` on the monad — putting a value in a context and gluing computation together.

`Free` is a recursive structure where each subsequent computation can access the result of a previous computation. This is all we need to build composable programs using plain data structures that are free to interpretation. Let’s return to this example:

```scala
FlatMap(Tell("Greetings!"), (_) => 
  FlatMap(Ask("What is your name?"), (name) => 
    Return(Tell(s"Hi $name!"))))
```

We want to end up having this kind of structure but it would be awkward to write programs by directly creating data structures. Besides, scala has a nice syntax to make it easier — “for comprehension”. This is how I’d like to see our program:

```scala
val program = for { 
  _    <- Tell("Greetings!")
  name <- Ask("What is your name?")
  _    <- Tell("Hi $name!")
} yield ()
```

To make that work we need two things:

* `Free` has to be a `Monad` (compiler will look for `flatMap` and `map` defined on Free trait)
* A way to construct values of `Free[UserInteraction, A]`. We want `flatMap` to be called on `Free`, not on `UserInteraction` (which is not a `Monad` and that’s the point)

## 💡 Free as a Monad

“For comprehension” is desugared into a sequence of `flatMap` calls ending up with `map`. They have to be defined on the `Free` trait:

```scala
sealed trait Free[F[_], A] {
  
  def flatMap[B](f: A => Free[F, B]): Free[F, B] = this match {
    case Return(a) => f(a)
    case FlatMap(sub, cont) => FlatMap(sub, cont andThen (_ flatMap f))
  }
  
  def map[B](f: A => B): Free[F, B] = flatMap(a => Return(f(a)))
  
}
```

`flatMap` has to deal with two cases: `Return` is simply about applying the `f` to the inner `a`, `FlatMap` is about creating the same `FlatMap` whose continuation created by composing self cont with the result of `flatMaping` `f`. Sounds complicated, sometimes its just better to follow the types and try to implement it yourself until it “clicks”.

Now we can do something like this:

```scala
// say we have 
// firstProgram: Free[UserInteraction, String]
// secondProgram: Free[UserInteraction, String]

val program = for { 
  _ <- firstProgram
  _ <- secondProgram
} yield ()
```

## 🏗 Creating Free stuff

How do we actually construct a `Free[UserInteraction, A]`? There has to be a function:

```scala
def toFree[A](fa: UserInteraction[A]): Free[UserInteraction, A] = FlatMap(fa, Return.apply)
```

`toFree` takes a `Tell` or `Ask` and returns a free monad constructed by `FlatMaping` the value with a function that will just yield the result. That allows us to do the following:

```scala
val program = for {
  _ <-    toFree(Tell("Hello!"))
  name <- toFree(Ask("What is your name?"))
  _ <-    toFree(Tell(s"Hi, $name"))
} yield ()
```

There is a trick to make the syntax nicer and not having to write `toFree` every time — make `toFree` available for implicit conversions:

```scala
// vvvvv
implicit def toFree[A](fa: UserInteraction[A]): Free[UserInteraction, A] = 
    FlatMap(fa, Return.apply)

val program = for {
  _ <-    Tell("Hello!")
  name <- Ask("What is your name?")
  _ <-    Tell(s"Hi, $name")
} yield ()
```

Now thanks to `Free` being a `Monad` and having a way to implicitly create `Free` from `Tell` and `Ask` our program looks exactly how we wanted to write it in the first place.

## 🏋️‍♀️ Do you even lift, Free?

Looking at `toFree` signature we can spot an improvement — abstracting over higher order type constructor it can be written in a more generic way, not specific `toUserInteraction`:

```scala
// also liftF is a usual name for creating Free
implicit def liftF[F[_], A](fa: F[A]): Free[F, A] = FlatMap(fa, Return.apply)
```

The problem here is that now it matches concrete type constructors:

```scala
//           vvvv Tell, but we want UserInteraction
val ua: Free[Tell, String] = liftF(Tell("Hello"))
```

We would run into compilation errors trying to compile for comprehension. The workaround is quite simple:

1. Define a type alias fixing `UserInteraction` to be a generator for `Free` which will represent the program’s DSL
2. Create functions to construct DSL by lifting our algebra types

```scala
// generalized liftF function
implicit def liftF[F[_], A](fa: F[A]): Free[F, A] = FlatMap(fa, Return.apply)

// Algebra type
type InteractionDsl[A] = Free[UserInteraction, A]

// "Smart" constructors
def tell[A](str: A): InteractionDsl[A] = liftF(Tell(str))

def ask[A](answer: A): InteractionDsl[A] = liftF(Ask(answer))

// Usage
val program: Free[UserInteraction, Unit] = for {
  _ <-    tell("Hello!")
  name <- ask("What is your name?")
  _ <-    tell(s"Hi, $name")
} yield ()
```

## 🏃 Running the program

So now we have a nice syntax and data structure representing our program, how do we actually run it? Similar to when we had just a `List` of instructions we can fold the `Free` executing each instruction. So currently our goal is: given program `Free[F, A]` traverse the `Free` structure evaluating each step and threading results to subsequent computations.

But first, lets start off simple with implementing `runInteractionDSL` for our concrete program `InteractionDsl[A]` and generalize to a `Free[F, A]` later. The first obvious thing to do is to pattern match on the argument:

```scala
def runInteractionDSL[A](prg: InteractionDsl[A]): A = prg match {
  case Return(a) => a
  case FlatMap(sub, cont) => ??? // how to execute sub?
}
```

How to run sub which is `UserInteraction[A]`? This can be either `Tell[A]` or `Ask[A]`, both of them has different meaning and there is no direct way to run them. We can use our previously implemented `execute` function to run sub operations, getting result back, feeding it back to `cont` and recursively call `runInteractionDSL`:

```scala
// Execution can be impure, side effects are OK here
def execute[A](ui: UserInteraction[A]): A = ui match {
  case Tell(str) =>
    println(str)
    str
  case Ask(question) =>
    println(question)
    val answer = scala.io.StdIn.readLine()
    answer.asInstanceOf[A]
}

def runInteractionDSL[A](prg: InteractionDsl[A]): A = prg match {
  case Return(a) => a
  case FlatMap(sub, cont) => {
    val result = execute(sub)
    runInteractionDSL(cont(result))
  }
}

runInteractionDSL(program)
```

## 📚 Generalizing over the algebra

Lets make a step towards more generic implementation. The `runFree` should be able to execute any `Free[F, A]`, not just `InteractionDsl[A]`(e.g. `Free[UserInteraction, A]`).

```scala
// Won't compile
def runFree[F[_], A](prg: Free[F, A]): A = prg match {
  case Return(a) => a
  case FlatMap(sub, cont) => {
    val result = execute(sub) // <-- a problem
    runFree(cont(result))
  }
}
```

The `execute` function is specific to `UserInteraction` and it’s not applicable to `Free[F, A]`. It has to be a generic function that by given `F[A]` will give us `A` back and possibly do some effects. Also, it has to be provided by the user and passed into `runFree`:

```scala
sealed trait Executor[F[_]] {
  def exec[A](fa: F[A]): A
}

def runFree[F[_], A](prg: Free[F, A], executor: Executor[F]): A = prg match {
  case Return(a) => a
  case FlatMap(sub, cont) => {
    val result = executor.exec(sub)
    runFree(cont(result), executor)
  }
}

// Usage
val consoleExec = new Executor[UserInteraction] {
  override def exec[A](fa: UserInteraction[A]) = fa match {
    case Tell(str) =>
      println(str)
      str
    case Ask(question) =>
      println(question)
      val answer = scala.io.StdIn.readLine()
      answer.asInstanceOf[A]
  }
}

runFree(program, consoleExec)
```

That’s pretty much it, we can run any `Free[F, A]` by providing a program to run and a custom “executor” that interprets our algebra. Looks great, but we can do even better.

## 🐛 Natural transformation 🦋

There is one more thing that can be generalized and that our “executor” function. Looking at the signature `F[A] => A `is actually a special case of functor transformation `F[A] => Id[A]` where `Id` is just a type alias for `A` (`type Id[A] = A`). This kind of transformation between functors is called “natural transformation”:

```scala
sealed trait NaturalTransformation[F[_], G[_]] {
  def transform[A](fa: F[A]): G[A] // <-- G[A] instead of just A
}
// Other known names are FunctionK or ~>
```

Natural transformation is just a function that maps values in one context to another (`F[A] ~> G[A]`). In our case it will map `UserInteraction` data types to an `Id`:

```scala
sealed trait NaturalTransformation[F[_], G[_]] {
  def transform[A](fa: F[A]): G[A]
}

// Won't compile
def runFree[F[_], G[_], A](prg: Free[F, A], nt: NaturalTransformation[F, G]): A = prg match {
  case Return(a) => a
  case FlatMap(sub, cont) => {
    val transformed = nt.transform(sub)
    runFree(cont(transformed), nt) // <- hmmm
  }
}
```

Previously `Executor` returned `A` which we could directly pass into continuation but now it returns `G[A]`. We can resolve it by requiring `G` to be a `Monad` and use `flatMap` to thread inner `A` into `cont` and return `G[A]`. The return value of `runFree` has to be `G[A]` instead of `A` as well:

```scala
// We could define Monad trait and monad instance for Id ourselves
// but since it's not critical for this article and for keeping 
// things simple let's just use Cats 
import cats.{Id, Monad}

sealed trait NaturalTransformation[F[_], G[_]] {
  def transform[A](fa: F[A]): G[A]
}
  
def runFree[F[_], G[_], A]
  (prg: Free[F, A], nt: NaturalTransformation[F, G])
  (implicit M: Monad[G]): G[A] =
  prg match {
    case Return(a) => Monad[G].pure(a)
    case FlatMap(sub, cont) => {
      val transformed = nt.transform(sub)
      Monad[G].flatMap(transformed) { a => runFree(cont(a), nt) }
    }
}

val consoleIO = new NaturalTransformation[UserInteraction, Id] {
  override def transform[A](fa: UserInteraction[A]) = fa match {
    case Tell(str) =>
      println(str)
      str
    case Ask(question) =>
      println(question)
      val answer = scala.io.StdIn.readLine()
      answer.asInstanceOf[A] 
  }
}
  
runFree(program, consoleIO)
```

Yay, finally we’re done. The real implementation that you might find in Cats is a bit different but most of that is about syntax and style — everything regarding `Free` is defined within the trait, `NaturalTransformation` is `calledFunctionK`, `transform` is `apply`, `run` is `foldMap` with curried arguments. The idea stays the same and hopefully our step by step approach served as a relatively simple introduction to the `Free` concept.

--- 

## Refereneces

* Rúnar Bjarnason’s [Composable application architecture with reasonably priced monads](https://www.youtube.com/watch?v=M258zVn4m2M)
* Rob Norris’ [Programs as Values: JDBC Programming with Doobie](https://www.youtube.com/watch?v=M5MF6M7FHPo)
*John de Goes’ [Move Over Free Monads: Make Way for Free Applicatives!](https://www.youtube.com/watch?v=H28QqxO7Ihc)