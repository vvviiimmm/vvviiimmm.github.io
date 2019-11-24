---
title: "Catamorphisms and F-Algebras"
categories:
  - Markup
tags:
  - html
  - markup
  - post
  - title
---

![image-center]({{ site.url }}{{ site.baseurl }}/assets/images/posts/cata/header.png){: .align-center}

> So I often hear the words “catamorphism” and “recursion schemes”. What is that about?
{: .notice}

Catamorphisms (or **cata**) are generalizations of the concept of a fold in functional programming. Given an F-Algebra and a recursive data structure a catamorphism will produce a value by recursively evaluating your data structure.

> What is an F-Algebra? Maybe you can show some code examples first?
{: .notice}

The setup is not that straightforward so let’s start simple. Let’s say you have the following data structure to represent an expression:

```haskell
data Expression = Value Int 
    | Add Expression Expression
    | Mult Expression Expression deriving Show
```

```scala
sealed trait Expression

case class Value(v: Int) extends Expression
case class Add(e1: Expression, e2: Expression) extends Expression
case class Mult(l: Expression, e2: Expression) extends Expression
```

And a simple expression can look like this:

```haskell
-- (1 + 2) * 3
expr = Mult (Add (Value 1) (Value 2)) (Value 3)
```

```scala
// (1 + 2) * 3
val expr = Mult(Add(Value(1), Value(2)), Value(3))
```

Having just an expression data structure is useless, of course you’ll need a function to evaluate and get the result:

```haskell
evalExpr :: Expression -> Int
evalExpr (Value v) = v
evalExpr (Add e1 e2) = evalExpr e1 + evalExpr e2
evalExpr (Mult e1 e2) = evalExpr e1 * evalExpr e2
```

```scala
def evalExpr(expr: Expression): Int = expr match {
  case Value(v) => v
  case Add(e1, e2) => evalExpr(e1) + evalExpr(e2)
  case Mult(e1, e2) => evalExpr(e1) * evalExpr(e2)
}
```

This is where catamorphism generalization comes in:

```haskell
notReallyCata :: (Expression -> Int) -> Expression -> Int
notReallyCata evaluator e = evaluator e -- purposly not Eta reducing
```

```scala
def notReallyCata(evaluator: (Expression) => Int, expr: Expression): Int = evaluator(expr)
```

> 🤨, but what’s the point? You just applying an argument to a function?
{: .notice}

That’s because it’s `notReallyCata`. The real `cata` function is generic and does not depend on any particular data structure or evaluator function. See, creation of a recursive data structure and then folding over it is a common pattern that `cata` tries to generalize.
> Ok, then how the real cata looks like?

```haskell
type Algebra f a = f a -> a 
newtype Mu f = InF { outF :: f (Mu f) } 
cata :: Functor f => Algebra f a -> Mu f -> a 
cata f = f . fmap (cata f) . outF
```

> 🤯
{: .notice}

That’s why we started with `notReallyCata`. We’ll break down the implementation later until it clicks. But now let’s continue with our `Expression` example. First, we need to get rid of recursion by introducing a type parameter:

```haskell
data ExpressionF a = ValueF Int
    | AddF a a
    | MultF a a deriving Show
```

```scala
sealed trait ExpressionF[A]

case class ValueF[A](v: Int) extends ExpressionF[A]
case class AddF[A](e1: A, e2: A) extends ExpressionF[A]
case class MultF[A](e1: A, e2: A) extends ExpressionF[A]
```

All references to `Expression` are replaced with `a` type parameter so the data structure is no longer recursive.

> Why is there an `F` at the end of type constructors?
{: .notice}

Glad you asked — that’s a hint that `ExpressionF` can be a `Functor`:

```haskell
instance Functor ExpressionF where
    fmap _ (ValueF a) = ValueF a
    fmap f (AddF e1 e2) = AddF (f e1) (f e2)
    fmap f (MultF e1 e2) = MultF (f e1) (f e2)
```

```scala
import cats.Functor

implicit object ExpressionFunctor extends Functor[ExpressionF] {
  override def map[A, B](fa: ExpressionF[A])(f: A => B): ExpressionF[B] = fa match {
    case ValueF(v)    => ValueF(v)
    case AddF(e1, e2) => AddF(f(e1), f(e2))
    case MultF(e1, e2) => MultF(f(e1), f(e2))
  }
}
```

Nothing fancy, just applying some function to the wrapped value preserving stucture.

> Not sure why we need that 🤔
{: .notice}

It doesn’t makes sense now but it will a bit later. Now, the way we create our expression haven’t changed (except for constructor names):

```haskell
expr =  Mult  (Add  (Value 1)  (Value 2))  (Value 3)
exprF = MultF (AddF (ValueF 1) (ValueF 2)) (ValueF 3)
```

```scala
val expr  = Mult (Add (Value(1),  Value(2)),  Value(3))
val exprF = MultF(AddF(ValueF(1), ValueF(2)), ValueF(3))
```

But the resulting type is different:

```haskell
expr :: Expression
expr =  Mult  (Add  (Value 1)  (Value 2))  (Value 3)

exprF :: ExpressionF (ExpressionF (ExpressionF a))
exprF = MultF (AddF (ValueF 1) (ValueF 2)) (ValueF 3)
```

```scala
val expr:  Expression                                 = Mult (Add (Value(1),  Value(2)),  Value(3))
val exprF: ExpressionF[ExpressionF[ExpressionF[Int]]] = MultF(AddF(ValueF(1), ValueF(2)), ValueF(3))
```

`expr` collapses everything into a single `Expression` while `exprF` encodes information about the nesting level of our expression tree. Speaking about evaluation, this is how we can go about implementing eval for `ExpressionF`:

```haskell
evalExprF :: ExpressionF Int -> Int
evalExprF (ValueF v) = v
evalExprF (AddF e1 e2) = e1 + e2
evalExprF (MultF e1 e2) = e1 * e2
```

```scala
def evalExprF(e: ExpressionF[Int]): Int = e match {
    case ValueF(v) => v
    case AddF(e1, e2) => e1 + e2
    case MultF(e1, e2) => e1 * e2
}
```

The main difference with original `evalExpr` is that we don’t have recursive call to `evalExprF` (`ExpressionF` is not recursive, remember?). It also means that our evaluator can work only with a **single level** expression:

```haskell
fourtyTwo = evalExprF (ValueF 42)
five = evalExprF (AddF 2 3)
six = evalExprF (MultF 2 3)
```

```scala
val fourtyTwo = evalExprF(ValueF(42))
val five = evalExprF(AddF(2, 3))
val six = evalExprF(MultF(2, 3))
```

And won’t compile on this:

```haskell
nestedExpr :: ExpressionF (ExpressionF Int)
nestedExpr = AddF (ValueF 2) (ValueF 3)
wontCompile = evalExprF nestedExpr
```

```scala
val nestedExpr: ExpressionF[ExpressionF[Int]] = AddF(ValueF(2), ValueF(3))
val wontCompile = evalExprF(nestedExpr)
```

Simply because `exprF` expepects `ExpressionF` Int and we’re shoving `ExpressionF (ExpressionF Int)`.

To make it work we could define another evaluator:

```haskell
-- Single level
evalExprF :: ExpressionF Int -> Int
evalExprF (ValueF v) = v
evalExprF (AddF e1 e2) = e1 + e2
evalExprF (MultF e1 e2) = e1 * e2

-- 2 Level, reusing evalExprF defined above
evalExprF2 :: ExpressionF (ExpressionF Int) -> Int
evalExprF2 (AddF e1 e2) = evalExprF e1 + evalExprF e2
evalExprF2 (MultF e1 e2) = evalExprF e1 * evalExprF e2
```

```scala
// Single level
def evalExprF(e: ExpressionF[Int]): Int = e match {
  case ValueF(v) => v
  case AddF(e1, e2) => e1 + e2
  case MultF(e1, e2) => e1 * e2
}

// 2 Level, reusing evalExprF defined above
def evalExprF2(e: ExpressionF[ExpressionF[Int]]): Int = e match {
  case ValueF(v) => v
  case AddF(e1, e2) => evalExprF(e1) + evalExprF(e2)
  case MultF(e1, e2) => evalExprF(e1) * evalExprF(e2)
}
```

> Looks kinda ad hoc, what if you have deeply nested expressions?
{: .notice}

Yes, for arbitrary nested expression this approach is not scalable — each additional nesting level requires you to write specialized function.

There is a way to generalize this nesting with a new type:

```haskell
newtype Fix f = Fx (f (Fix f))

unfix :: Fix f -> f (Fix f)
unfix (Fx x) = x
```

```scala
final case class Fix[F[_]](unFix: F[Fix[F]])
```

> Fix? Looks like a recursive data structure that doesn’t do much. How is it useful?
{: .notice}

Let’s first look at the expression before the equals sign: indeed `Fix` is a recursive data structure that has one type parameter `f`. This parameter has kind `* -> *` e.g. it also takes a type parameter. For example, you can’t construct Fix providing Int or Bool, it has to be something like `Maybe`, `List` or… `ExpressionF`. This is why we introduced type parameter for `ExpressionF`. Next, after the equals sign we have a single type constructor `Fx` taking a single argument of type `f (Fix f)` which is basically an expression that constructs `f`'s value. In case of `Maybe` it would be `Maybe (Fix Maybe)` and then the whole thing is wrapped with `Fx` into type `Fix Maybe`.

The type signature is confusing to read at first because of type constructor can have the same name as the type itself plus self referencing. But there is not much more to it than just wrapping a higher order type into a data structure. Btw, `unfix` is an opposite to `Fx` and all it does is pattern matching on `Fx` and returns wrapped value, no big deal.

Now, we will replace every `ExpressionF` of our expression tree with `Fix ExpressionF`. Notice the difference in constructing expressions with and without `Fx` — they’re basically the same, except we need to prepend `Fx $`:

```haskell
-- (1 + 2) * (3 + 4)
regularExprF :: ExpressionF (ExpressionF (ExpressionF a))
regularExprF = MultF (AddF (ValueF 1) (ValueF 2)) (AddF (ValueF 3) (ValueF 4))

fixedExprF :: Fix ExpressionF
fixedExprF   = Fx $ MultF (Fx $ AddF (Fx $ ValueF 1) (Fx $ ValueF 2)) (Fx $ AddF (Fx $ ValueF 3) (Fx $ ValueF 4))
```

```scala
val regularExprF: ExpressionF[ExpressionF[ExpressionF[Int]]] =
  MultF(
    AddF(
      ValueF(1),
      ValueF(2)),
    AddF(
      ValueF(3),
      ValueF(4)))

val fixedExprF: Fix[ExpressionF] =
  Fix(MultF(
    Fix(AddF(
      Fix(ValueF(1)),
      Fix(ValueF(2)))),
    Fix(AddF(
      Fix(ValueF(1)),
      Fix(ValueF(2))))))
```

The resulting type of a ‘fixed’ version is `Fix ExpressionF` so we’re back to a recursive representation, but now we have to use `unfix` function to get our non recursive data structure back.

> What are the benefits of having `Fix`? Looks like it’s the same approach as original `Expression` type but now we have this weird `Fix` and `unfix` nonsense?
{: .notice}

Yes, but we’re trying to generalize the process of folding, it requires introduction of additional abstractions, like `Fix` and `Algebra` that we’ll discuss later. Bear with me, it should make more sense later.

So we have our ‘fixed’ data structure, how would evaluation function look like?

```haskell
evalFixedExprF :: Fix ExpressionF -> Int
evalFixedExprF fixedExpr = ???
```

```scala
def evalFixedExprF(e: Fix[ExpressionF]): Int = ???
```

Given a `Fix ExpressionF` the only thing we can do with it is calling `unfix` which produces `ExpressionF (Fix ExpressionF)`:

```haskell
evalFixedExprF :: Fix ExpressionF -> Int
evalFixedExprF fixedExpr = unfix fixedExpr ?? and something
```

```scala
def evalFixedExprF(e: Fix[ExpressionF]): Int = e.unFix ???
```

The returned `ExpressionF` can be one of our `ValueF`, `AddF` or `MultF` having a `Fix ExpressionF` as their type parameter. It makes sense to do pattern matching and decide what to do next:

```haskell
evalFixedExprF :: Fix ExpressionF -> Int
evalFixedExprF fixedExpr = case unfix fixedExpr of
    ValueF i -> i
    AddF fixE1 fixE2 -> evalFixedExprF fixE1 + evalFixedExprF fixE2
    MultF fixE1 fixE2 -> evalFixedExprF fixE1 * evalFixedExprF fixE2
    
-- looks familiar?
evalExpr :: Expression -> Int
evalExpr (Value v) = v
evalExpr (Add e1 e2) = evalExpr e1 + evalExpr e2
evalExpr (Mult e1 e2) = evalExpr e1 * evalExpr e2
```

```scala
def evalFixedExprF(e: Fix[ExpressionF]): Int = e.unFix match {
    case ValueF(v) => v
    case AddF(e1, e2) => evalFixedExprF(e1) + evalFixedExprF(e2)
    case MultF(e1, e2) => evalFixedExprF(e1) * evalFixedExprF(e2)
}

// looks familiar?
def evalExpr(expr: Expression): Int = expr match {
  case Value(v) => v
  case Add(e1, e2) => evalExpr(e1) + evalExpr(e2)
  case Mult(e1, e2) => evalExpr(e1) * evalExpr(e2)
}
```

Yes, it looks the same as our very first recursive evaluator for `Expression` with addition of having to unwrap the expression with `unfix`. So why bother with `Fix` anyway?

Here’s the key: we will re-use our original ‘fix-less’ evaluator for `ExpressionF` and somehow distribute it over the `Fix ExpressionF` stucture. So this should be a function taking two arguments — the evaluator and the structure to evaluate:

```haskell
almostCata :: (ExpressionF Int -> Int) -> Fix ExpressionF -> Int
almostCata evaluator expr = undefined
```

```scala
def almostCata(evaluator: (ExpressionF[Int] => Int), e: Fix[ExpressionF]): Int = ???
```

Let’s try figure out the implementation — the first logical thing to do is to use `unfix` to get `ExpressionF` and then maybe pass it to `evaluator`:

```haskell
almostCata :: (ExpressionF Int -> Int) -> Fix ExpressionF -> Int
almostCata evaluator expr = evaluator $ unfix expr -- won't compile
```

```scala
def almostCata(evaluator: (ExpressionF[Int] => Int), e: Fix[ExpressionF]): Int = 
  evaluator(e.unFix) // won't compile
```

Obviously this doesn’t work, `evaluator` expects `ExpressionF Int` and not `ExpressionF (Fix ExpressionF)`. By the way, remember that `ExpressionF` is a `Functor`? This is where it gets handy — we can use `fmap` to apply the same process to the inner level of our expression tree:

```haskell
almostCata :: (ExpressionF Int -> Int) -> Fix ExpressionF -> Int
almostCata evaluator expr = evaluator $ fmap (almostCata evaluator) (unfix expr)
```

```scala
def almostCata(evaluator: (ExpressionF[Int] => Int))(e: Fix[ExpressionF]): Int =
   evaluator(Functor[ExpressionF].map(e.unFix)(almostCata(evaluator)))
```

Take a moment and think about what happens: we’re passing a recursive function `almostCata evaluator` into the `fmap`. If the current expression is `AddF` or `MultF` then this function will be passed one level deeper and `fmap` will be called again. This will happen until we reach `ValueF`, fmapping over `ValueF` returns value of type `ExpressionF Int` and that’s exactly what our evaluator function accepts.

By looking at `almostCata` we can see that it doesn’t really have anything specific to `ExpressionF` or `Int` type and theoretically can be generalized with some type parameter `f`. The only constraint should be having a `Functor` instance for `f`, because we’re using `fmap`:

```haskell
cata :: Functor f => (f a -> a) -> Fix f -> a
cata alg expr = alg $ fmap (cata alg) (unfix expr)
```

```scala
def cata[F[_], A](alg: (F[A] => A))(e: Fix[F])(implicit F: Functor[F]): A =
  alg(F.map(e.unFix)(cata(alg)))
```

And that’s the final version of `cata`. Here’s the full implementation with some usage examples:

```haskell
-- Non recursive data structure
data ExpressionF a = ValueF Int
    | AddF a a
    | MultF a a deriving Show

-- Functor instance for it
instance Functor ExpressionF where
    fmap _ (ValueF a) = ValueF a
    fmap f (AddF e1 e2) = AddF (f e1) (f e2)
    fmap f (MultF e1 e2) = MultF (f e1) (f e2)

-- Evaluator function
evalExprF :: ExpressionF Int -> Int  
evalExprF (ValueF v) = v
evalExprF (AddF e1 e2) = e1 + e2
evalExprF (MultF e1 e2) = e1 * e2

-- Fix data structure
newtype Fix f = Fx (f (Fix f))
unfix :: Fix f -> f (Fix f)
unfix (Fx x) = x

-- Catamporphism
cata :: Functor f => (f a -> a) -> Fix f -> a
cata alg expr = alg $ fmap (cata alg) (unfix expr)

-- Example expression and evaluation
someExpression   = Fx $ MultF (Fx $ AddF (Fx $ ValueF 1) (Fx $ ValueF 2)) (Fx $ AddF (Fx $ ValueF 3) (Fx $ ValueF 4))
twentyOne = cata evalExprF someExpression
```

```scala
// Non recursive data structure
sealed trait ExpressionF[A]

case class ValueF[A](v: Int) extends ExpressionF[A]
case class AddF[A](e1: A, e2: A) extends ExpressionF[A]
case class MultF[A](e1: A, e2: A) extends ExpressionF[A]

// Functor instance for it
implicit object ExpressionFunctor extends Functor[ExpressionF] {
  override def map[A, B](fa: ExpressionF[A])(f: A => B): ExpressionF[B] = fa match {
    case ValueF(v)    => ValueF(v)
    case AddF(e1, e2) => AddF(f(e1), f(e2))
    case MultF(e1, e2) => MultF(f(e1), f(e2))
  }
}

// Evaluation function
def evalExprF(e: ExpressionF[Int]): Int = e match {
  case ValueF(v) => v
  case AddF(e1, e2) => e1 + e2
  case MultF(e1, e2) => e1 * e2
}

// Fix data structure
final case class Fix[F[_]](unFix: F[Fix[F]])

// Catamorphism
def cata[F[_], A](alg: (F[A] => A))(e: Fix[F])(implicit F: Functor[F]): A =
  alg(F.map(e.unFix)(cata(alg)))

// Example expression and evaluation
val someExpression: Fix[ExpressionF] =
  Fix(MultF(
    Fix(AddF(
      Fix(ValueF(1)),
      Fix(ValueF(2)))),
    Fix(AddF(
      Fix(ValueF(3)),
      Fix(ValueF(4))))))

val twentyOne = cata(evalExprF)(someExpression)
```
> I guess that’s cool. But why tho?

A lot of concepts in category theory and functional programming are pretty abstract and sometimes it’s hard to find immediate practical application for certain idea. But looking for abstractions and generalizations is useful for finding patterns and elegant solutions to problems that otherwise require ad-hoc implementation.

By the way, by generalizing our `ExpressionF -> Int` function to `Functor f => (f a -> a)` we discovered another important concept called **F-Algebra**. Basically F-Algebra is a triple of functor `f`, some type `a` and evaluator function `f a -> a`. Note that `a` here not polymorphic — it has to be a concrete type, like `Int` or `Bool` and it’s called a **carrier type**. For any endo-functor `f` you can create multiple F-Algebra’s based on it. Take our expressions example — endo-functor `f` is `ExpressionF`, a is `Int` and evaluator is `evalExprF`. But we can change the carrier type and produce more algebras:

```haskell
type Algebra f a = f a -> a

algebra0 :: Algebra ExpressionF Int -- or ExpressionF Int -> Int
algebra0 (ValueF v) = v
algebra0 (AddF e1 e2) = e1 + e2
algebra0 (MultF e1 e2) = e1 * e2

algebra1 :: Algebra ExpressionF String
algebra1 (ValueF v) = show v
algebra1 (AddF e1 e2) = "(" ++ e1 ++ " " ++ e2 ++ ")"
algebra1 (MultF e1 e2) = e1 ++ e2

algebra2 :: Algebra ExpressionF Bool
algebra2 (ValueF v) = True
algebra2 (AddF e1 e2) = e1 || e2
algebra2 (MultF e1 e2) = e1 && e2
```

```scala
type Algebra[F[_], A] = F[A] => A

val algebra0: Algebra[ExpressionF, Int] = {
  case ValueF(v) => v
  case AddF(e1, e2) => e1 + e2
  case MultF(e1, e2) => e1 * e2
}

val algebra1: Algebra[ExpressionF, String] = {
  case ValueF(v) => v.toString
  case AddF(e1, e2) => "(" ++ e1 ++ " " ++ e2 ++ ")"
  case MultF(e1, e2) => e1 ++ e2
}

val algebra2: Algebra[ExpressionF, Boolean] = {
  case ValueF(_) => true
  case AddF(e1, e2) => e1 || e2
  case MultF(e1, e2) => e1 && e2
}
```

> That’s just different evaluators that can be passed into `cata`, right?
{: .notice}

Yes, we’re picking different carrier types and choosing our implementation. But there the trick — there is a mother of all evaluators that we can create by picking our carrier type to be… `Fix ExprF`.

```haskell
-- just in case:
newtype Fix f = Fx (f (Fix f))

initialAlgebra :: ExpressionF (Fix ExpressionF) -> Fix ExpressionF
initialAlgebra = ?
```

```scala
// just in case
final case class Fix[F[_]](unFix: F[Fix[F]])

//                                       vvvvvvvvvvvvvvvv
val initialAlgebra: Algebra[ExpressionF, Fix[ExpressionF]] = ???
```
> Evaluating to `Int` or `Bool` totally makes sense but what would this `initialAlgebra` evaluate? When do I need to have `Fix` of something as a result of my evaluator?
{: .notice}

Of course you won’t write something like that yourself, just want to show you the deeper meaning behind f-algebras and cata. In fact, we already have an implementation for such evaluator and thats exactly `Fx` constructor:

```haskell
initialAlgebra :: ExpressionF (Fix ExpressionF) -> Fix ExpressionF
initialAlgebra = Fx
```

```scala
val initialAlgebra: Algebra[ExpressionF, Fix[ExpressionF]] = Fix[ExpressionF]
```

> Wait, `Fx` is an evaluator? That’s crazy.
{: .notice}

Yes and it does the most simple thing you can do — save the expession into a data structure. While all other evaluators (`algebra0`, `algebra1`) produced some value by reducing the expression (like doing sum or concatenation) Fx just wraps the expression without loosing any data.

This is why we introduced `Fix` in the first place — you first evaluate your original data structure with `Fx` into initial algebra `Fix f` and then using `cata` the ‘real’ evaluation happens by fmaping your concrete evaluator over inital algebra.

From category theory point of view, all algebras based on the same endo-functor form a category. This category has an initial object which is our initial algebra created by picking the carrier type as `Fix f`. There are some great blog posts by Bartosz Milewski that I highly recommend checking out if you want to get deep categorical understanding.

> It’s still pretty hard to comprehend, I don’t think I fully understand the concept
{: .notice}

It’s always better to do hands on: try re-implementing `Fix` and `cata` on your own, think about possible data structures and algebras. For example, a `String` can be represented recursively (as a `Char` head and tail of `String`), the length of a string can be computed with `cata`. Here’s some great resources for further reading:

* [Understanding F-Algebras](https://www.schoolofhaskell.com/user/bartosz/understanding-algebras) and slightly different [F-Algebras](https://bartoszmilewski.com/2017/02/28/f-algebras/) by [Bartosz Milewski](https://bartoszmilewski.com)

* [Catamorphisms in 15 minutes](http://chrislambda.github.io/blog/2014/01/30/catamorphisms-in-15-minutes/) by Chris Jones

* [Pure Functional Database Programming with Fixpoint Types](https://www.youtube.com/watch?v=7xSfLPD6tiQ) by Rob Norris

* [Catamorphisms](https://wiki.haskell.org/Catamorphisms) on Haskell wiki

* [Practical recursion schemes](https://jtobin.io/practical-recursion-schemes) by [Jared Tobin](https://jtobin.io)
