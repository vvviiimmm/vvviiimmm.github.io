---
title: "The Evolution of a Scala Programmer"
classes: wide
categories:
  - scala
tags:
  - functional programming
  - scala
---

## 17 different ways to calculate a factorial.

![image-center]({{ site.url }}{{ site.baseurl }}/assets/images/posts/scala-evolution/header.png){: .align-center}

Disclaimer: this article is a parody inspired by the [Haskell version](https://www.willamette.edu/~fruehr/haskell/evolution.html), please don’t take it too seriously. However, I believe that some examples actually have some value and can be used as a reference.

---

### Table of Contents
* [Beginner Scala programmer](https://vvviiimmm.github.io/scala/scala-evolution/#beginner-scala-programmer)
* [Beginner functional Scala programmer](https://vvviiimmm.github.io/scala/scala-evolution/#beginner-functional-scala-programmer)
* [Scala programmer circa 2011](https://vvviiimmm.github.io/scala/scala-evolution/#scala-programmer-circa-2011)
* [Free monadic Scala programmer](https://vvviiimmm.github.io/scala/scala-evolution/#free-monadic-scala-programmer)
* [Freak monad Scala programmer](https://vvviiimmm.github.io/scala/scala-evolution/#freak-monadic-scala-programmer)
* [Tagless Final Scala programmer](https://vvviiimmm.github.io/scala/scala-evolution/#tagless-final-scala-programmer)
* [John A De Goes](https://vvviiimmm.github.io/scala/scala-evolution/#john-a-de-goes)
* [Fabio Labella](https://vvviiimmm.github.io/scala/scala-evolution/#fabio-labella)
* [Akka Scala programmer](https://vvviiimmm.github.io/scala/scala-evolution/#akka-scala-programmer)
* [Spark Scala programmer](https://vvviiimmm.github.io/scala/scala-evolution/#spark-scala-programmer)
* [Java Scala programmer](https://vvviiimmm.github.io/scala/scala-evolution/#java-scala-programmer)
* [Trampolined Scala programmer](https://vvviiimmm.github.io/scala/scala-evolution/#trampolined-scala-programmer)
* [Peano ADT Scala programmer](https://vvviiimmm.github.io/scala/scala-evolution/#peano-adt-scala-programmer)
* [Type level Scala programmer](https://vvviiimmm.github.io/scala/scala-evolution/#type-level-scala-programmer)
* [Shapeless Scala programmer](https://vvviiimmm.github.io/scala/scala-evolution/#shapeless-scala-programmer)
* [Fix Scala programmer](https://vvviiimmm.github.io/scala/scala-evolution/#fix-scala-programmer)
* [Paramorphism Scala programmer](https://vvviiimmm.github.io/scala/scala-evolution/#paramorphism-scala-programmer)
* [Apo-futu-hylomorphism Scala programmer](https://vvviiimmm.github.io/scala/scala-evolution/#apo-futu-hylomorphism-scala-programmer)

---

## Beginner Scala programmer

Coming from Java likes the fact that semicolons are optional. And `return`. And type declarations.

```scala
object Main extends App {
  print("Please enter a number: ")

  val number = scala.io.StdIn.readInt()
  var total = 1

  for (x <- 1 to number)
    total = total * x

  println("Factorial of " + number.toString + " is " + total.toString)
}
```

## Beginner functional Scala programmer

Forced by his teammates to use this weird IO thing. Can’t get approval for his small PR for weeks. Unsure if it was worth quitting his high paying Java position.

```scala
import cats.effect.IO

import scala.util.{Failure, Success, Try}

object Main extends App {

  /**
    * Pure functions, we're good!
    */
  def putStr(msg: String): IO[Unit] = IO(print(msg))

  def getStr: IO[String] = IO(scala.io.StdIn.readLine())

  def validateInt(str: String): IO[Int] = IO(str.toInt)

  def factorial(n: Int): BigInt = Range.BigInt(1, n, 1).product

  /**
    * It's the end of the world now, no one cares.
    */
  putStr("Please enter a number: ").unsafeRunSync()
  val input = getStr.unsafeRunSync()
  Try(validateInt(input).unsafeRunSync()) match {
    case Success(n) =>
      putStr(s"Factorial of $n is ${factorial(n)}").unsafeRunSync()
    case Failure(_) => putStr("Invalid number").unsafeRunSync()
  }
}
```

## Scala programmer circa 2011

Traits and mixins! Believes in OOP and SOLID, likes cakes.

```scala
object Main extends App {

  trait Console {
    def printLine(msg: String): Unit
    def readLine: String
  }

  trait ConsoleComponent {
    val console: Console
  }

  trait ConsoleLive extends ConsoleComponent {
    lazy val console: Console = new Console {
      def printLine(msg: String): Unit = print(msg)
      def readLine: String = scala.io.StdIn.readLine()
    }
  }

  trait Validation {
    def validateInteger(str: String): Option[Int]
  }

  trait ValidationComponent {
    val validation: Validation
  }

  trait ValidationLive extends ValidationComponent {
    import scala.util.Try
    lazy val validation: Validation = new Validation {
      def validateInteger(str: String): Option[Int] = Try(str.toInt).toOption
    }
  }

  trait Calculator {
    def factorial(n: Int): BigInt
  }

  trait CalculatorComponent {
    val calculator: Calculator
  }

  trait CalculatorLive extends CalculatorComponent {
    lazy val calculator: Calculator = new Calculator {
      def factorial(n: Int): BigInt = {
        var total: BigInt = 1
        for (x <- 1 to n)
          total = total * x
        total
      }
    }
  }

  trait MyProgram {
    this: CalculatorComponent with ValidationComponent with ConsoleComponent =>

    def run(): Unit = {
      this.console.printLine("Please enter a number: ")
      val input = this.console.readLine
      val maybeNumber = this.validation.validateInteger(input)
      if (maybeNumber.isDefined) {
        val fact = this.calculator.factorial(maybeNumber.get)
        this.console.printLine(s"Factorial of ${maybeNumber.get} is $fact")
      } else
        console.printLine("Invalid number")
    }
  }

  val program = new MyProgram with CalculatorLive with ValidationLive with ConsoleLive

  program.run()
}
```

## Free monadic Scala programmer

Has a list of his favorite puns around the word `Free`, like “reasonably priced monads” or “`Free` is not for free”. Read all of Bartosz Milewski’s category theory installments. Favorite phrase: “`X`? But it’s just a `Y` in the category of `Z`!”

```scala
object Main extends App {

  case class Coproduct[F[_], G[_], A](run: Either[F[A], G[A]])

  trait Monad[F[_]] {
    def pure[A](x: A): F[A]
    def flatMap[A, B](fa: F[A])(f: A => F[B]): F[B]
  }

  object Monad {
    def apply[F[_]](implicit monad: Monad[F]): Monad[F] = monad
  }

  type Id[A] = A

  implicit val monadId: Monad[Id] = new Monad[Id] {
    def pure[A](x: A): Id[A] = x
    def flatMap[A, B](fa: Id[A])(f: A => Id[B]): Id[B] = f(fa)
  }

  trait NaturalTransformation[F[_], G[_]] { self =>
    def transform[A](fa: F[A]): G[A]

    def or[H[_]](f: NaturalTransformation[H, G])
      : NaturalTransformation[({ type f[x] = Coproduct[F, H, x] })#f, G] =
      new NaturalTransformation[({ type f[x] = Coproduct[F, H, x] })#f, G] {
        def transform[A](c: Coproduct[F, H, A]): G[A] = c.run match {
          case Left(fa)  => self.transform(fa)
          case Right(ha) => f.transform(ha)
        }
      }
  }

  sealed trait Free[F[_], A] {
    def flatMap[B](f: A => Free[F, B]): Free[F, B] = this match {
      case Return(a)          => f(a)
      case FlatMap(sub, cont) => FlatMap(sub, cont andThen (_ flatMap f))
    }
    def map[B](f: A => B): Free[F, B] = flatMap(a => Return(f(a)))
    def foldMap[G[_]: Monad](f: NaturalTransformation[F, G]): G[A] =
      this match {
        case Return(a) => Monad[G].pure(a)
        case FlatMap(fx, g) =>
          Monad[G].flatMap(f.transform(fx)) { a =>
            g(a).foldMap(f)
          }
      }
  }

  case class Return[F[_], A](a: A) extends Free[F, A]

  case class FlatMap[F[_], I, A](sub: F[I], cont: I => Free[F, A])
      extends Free[F, A]

  object Free {
    implicit def liftF[F[_], A](fa: F[A]): Free[F, A] =
      FlatMap(fa, Return.apply)

    def runFree[F[_], G[_], A](
        prg: Free[F, A],
        nt: NaturalTransformation[F, G])(implicit M: Monad[G]): G[A] =
      prg match {
        case Return(a) => Monad[G].pure(a)
        case FlatMap(sub, cont) => {
          val transformed = nt.transform(sub)
          Monad[G].flatMap(transformed) { a =>
            runFree(cont(a), nt)
          }
        }
      }
  }

  sealed trait Inject[F[_], G[_]] {
    def inj[A](sub: F[A]): G[A]
    def prj[A](sup: G[A]): Option[F[A]]
  }

  object Inject {
    implicit def injRefl[F[_]]: Inject[F, F] = new Inject[F, F] {
      def inj[A](sub: F[A]): F[A] = sub
      def prj[A](sup: F[A]) = Some(sup)
    }

    implicit def injLeft[F[_], G[_]] =
      new Inject[F, ({ type λ[α] = Coproduct[F, G, α] })#λ] {
        def inj[A](sub: F[A]) = Coproduct(Left(sub))
        def prj[A](sup: Coproduct[F, G, A]): Option[F[A]] = sup.run match {
          case Left(fa) => Some(fa)
          case Right(_) => None
        }
      }

    implicit def injRight[F[_], G[_], H[_]](implicit I: Inject[F, G]) =
      new Inject[F, ({ type f[x] = Coproduct[H, G, x] })#f] {
        def inj[A](sub: F[A]) = Coproduct(Right(I.inj(sub)))
        def prj[A](sup: Coproduct[H, G, A]): Option[F[A]] = sup.run match {
          case Left(_)  => None
          case Right(x) => I.prj(x)
        }
      }
  }

  def lift[F[_], G[_], A](f: F[A])(implicit I: Inject[F, G]): Free[G, A] =
    FlatMap(I.inj(f), Return(_: A))

  sealed trait Console[A]

  case class Print(msg: String) extends Console[Unit]

  case object ReadLine extends Console[String]

  sealed trait Validation[A]

  case class ValidateInteger(text: String) extends Validation[Option[Int]]

  sealed trait Calculation[A]
  case class Factorial(n: Int) extends Calculation[BigInt]

  class HasConsole[F[_]](implicit I: Inject[Console, F]) {
    def putStr(msg: String): Free[F, Unit] = lift(Print(msg))
    def getStr: Free[F, String] = lift(ReadLine)
  }

  class HasValidation[F[_]](implicit I: Inject[Validation, F]) {
    def validateInteger(str: String): Free[F, Option[Int]] =
      lift(ValidateInteger(str))
  }

  object HasConsole {
    implicit def instance[F[_]](implicit I: Inject[Console, F]): HasConsole[F] =
      new HasConsole[F]
  }

  object HasValidation {
    implicit def instance[F[_]](
        implicit I: Inject[Validation, F]): HasValidation[F] =
      new HasValidation[F]
  }

  val consoleInterpreter: NaturalTransformation[Console, Id] =
    new NaturalTransformation[Console, Id] {
      def transform[A](fa: Console[A]): Id[A] = fa match {
        case Print(str) =>
          print(str)
        case ReadLine =>
          scala.io.StdIn.readLine()
      }
    }

  val validationInterpreter: NaturalTransformation[Validation, Id] =
    new NaturalTransformation[Validation, Id] {
      import scala.util.Try
      def transform[A](fa: Validation[A]): Id[A] = fa match {
        case ValidateInteger(str) => Try(str.toInt).toOption
      }
    }

  /**
    * Alright, now let's actually write our program
    */
  def freeProgram[F[_]](implicit C: HasConsole[F],
                        V: HasValidation[F]): Free[F, Unit] = {
    def factorial(n: Int): BigInt = Range.BigInt(1, n, 1).product
    for {
      _ <- C.putStr("Please enter a number: ")
      input <- C.getStr
      maybeNumber <- V.validateInteger(input)
      _ <- maybeNumber.fold(C.putStr("Invalid number"))(n =>
        C.putStr(s"Factorial of $n is ${factorial(n)}"))
    } yield ()
  }
  
  val programInterpreter = consoleInterpreter or validationInterpreter

  type AppDSL[A] = Coproduct[Console, Validation, A]

  /**
    * Instantiate the program for AppDSL
    */
  val app: Free[AppDSL, Unit] = freeProgram[AppDSL]

  app.foldMap(programInterpreter)
}
```

## Freak monad Scala programmer

Tired of lifting and composing multiple algebras with coproducts, throws some type level magic in there. Spends all that free time (FreeK time?) on figuring out type checker’s cryptic error messages.

```scala
import cats.free.Free
import cats.~>
import freek.{:|:, DSL, NilDSL}

import freek._

object Main extends App {

  sealed trait Console[A]
  case class Print(msg: String) extends Console[Unit]
  case object ReadLine extends Console[String]

  sealed trait Validation[A]
  case class ValidateInteger(text: String) extends Validation[Option[Int]]

  sealed trait Calculation[A]
  case class Factorial(n: Int) extends Calculation[BigInt]

  /**
    * Program DSL is a coproduct of all the algebras
    */
  type PRG = Console :|: Validation :|: Calculation :|: NilDSL
  val PRG = DSL.Make[PRG]

  def program: Free[PRG.Cop, Unit] =
    for {
      _ <- Print("Please enter a number: ").freek[PRG]
      input <- ReadLine.freek[PRG]
      maybeNumber <- ValidateInteger(input).freek[PRG]
      _ <- maybeNumber.fold(Print("Invalid number").freek[PRG])(
        n =>
          Factorial(n)
            .freek[PRG]
            .flatMap(fact => Print(s"Factorial of $n is $fact").freek[PRG]))
    } yield ()

  type Id[A] = A

  /**
    * Interpreters for each algebra
    */
  val ConsoleInterpreter = new (Console ~> Id) {
    def apply[A](a: Console[A]): A = a match {
      case Print(msg) =>
        print(s"$msg")
      case ReadLine =>
        scala.io.StdIn.readLine()
    }
  }

  val ValidationInterpreter = new (Validation ~> Id) {
    import scala.util.Try
    def apply[A](a: Validation[A]): A = a match {
      case ValidateInteger(str) => Try(str.toInt).toOption
    }
  }

  val CalculationInterpreter = new (Calculation ~> Id) {
    def factorial(n: Int): BigInt = {
      def factorial_(n: Int, total: BigInt): BigInt =
        if (n <= 1)
          total
        else
          factorial_(n - 1, total * n)
      factorial_(n, 1)
    }
    def apply[A](a: Calculation[A]): A = a match {
      case Factorial(n) => factorial(n)
    }
  }

  /**
    * The program interpreter is a sum of all the interpreters
    */
  val interpreter = ConsoleInterpreter :&: ValidationInterpreter :&: CalculationInterpreter
  program.interpret(interpreter)
}
```

## Tagless Final Scala programmer

Has (monad) commitment issues. While texting all the words starting with `f` are autocorrected to `F[_]`. Thinks about switching to Haskell someday.

```scala
object Main extends App {

  trait Console[F[_]] {
    def putStr(msg: String): F[Unit]
    def getStr: F[String]
  }

  object Console {
    def apply[F[_]](implicit F: Console[F]): Console[F] = F
  }

  trait Calculator[F[_]] {
    def factorial(n: Int): F[BigInt]
  }

  object Calculator {
    def apply[F[_]](implicit F: Calculator[F]): Calculator[F] = F
  }

  trait Validation[F[_]] {
    def validateInt(str: String): F[Option[Int]]
  }

  object Validation {
    def apply[F[_]](implicit F: Validation[F]): Validation[F] = F
  }

  trait Monad[F[_]] {
    def pure[A](x: A): F[A]
    def flatMap[A, B](fa: F[A])(f: A => F[B]): F[B]
  }

  class IO[+A](val unsafeRun: () => A) { s =>
    def map[B](f: A => B): IO[B] = flatMap(f.andThen(IO(_)))
    def flatMap[B](f: A => IO[B]): IO[B] = IO(f(s.unsafeRun()).unsafeRun())
  }
  object IO {
    def apply[A](eff: => A): IO[A] = new IO(() => eff)
  }

  implicit class MOps[M[_], A](m: M[A])(implicit monad: Monad[M]) {
    def flatMap[B](f: A => M[B]): M[B] = monad.flatMap(m)(f)
    def map[B](f: A => B): M[B] = monad.flatMap(m)(f.andThen(monad.pure))
  }

  def program[F[_]: Monad: Console: Calculator: Validation]: F[Unit] = {
    for {
      _ <- Console[F].putStr("Please enter a number: ")
      input <- Console[F].getStr
      maybeInt <- Validation[F].validateInt(input)
      _ <- maybeInt.fold(Console[F].putStr("Invalid number"))(
        n =>
          Calculator[F]
            .factorial(n)
            .flatMap(fact => Console[F].putStr(s"Factorial of $n is $fact")))
    } yield ()
  }

  implicit val consoleIO: Console[IO] = new Console[IO] {
    def putStr(msg: String): IO[Unit] = IO(print(msg))
    def getStr: IO[String] = IO(scala.io.StdIn.readLine())
  }

  implicit val validationIO: Validation[IO] = new Validation[IO] {
    import scala.util.Try
    def validateInt(str: String): IO[Option[Int]] =
      IO(Try(str.toInt).toOption)
  }

  implicit val calculatorIO: Calculator[IO] = new Calculator[IO] {
    def factorial(n: Int): IO[BigInt] = IO(Range.BigInt(1, n, 1).product)
  }

  implicit val ioMonad: Monad[IO] = new Monad[IO] {
    def pure[A](x: A): IO[A] = IO(x)
    def flatMap[A, B](fa: IO[A])(f: A => IO[B]): IO[B] = fa.flatMap(f)
  }

  program[IO].unsafeRun()
}
```

## John A De Goes

Type-safe, composable, asynchronous, concurrent, high performance, resource-safe, testable, functional and resilient factorial.

```scala
import scalaz.zio.{DefaultRuntime, Fiber, Task, UIO, ZIO}

object Main extends App {

  object ConsoleIO {
    trait Service {
      def putStr(msg: String): Task[Unit]
      def getStr: Task[String]
    }
  }
  trait ConsoleIO {
    def consoleIO: ConsoleIO.Service
  }
  object consoleIO {
    def putStr(msg: String): ZIO[ConsoleIO, Throwable, Unit] =
      ZIO.accessM(_.consoleIO.putStr(msg))
    def getStr: ZIO[ConsoleIO, Throwable, String] =
      ZIO.accessM(_.consoleIO.getStr)
  }
  trait ConsoleIOLive extends ConsoleIO {
    def consoleIO: ConsoleIO.Service = new ConsoleIO.Service {
      def putStr(msg: String): Task[Unit] = Task.effect(print(msg))
      def getStr: Task[String] = Task.effect(scala.io.StdIn.readLine())
    }
  }
  object ConsoleIOLive extends ConsoleIOLive

  object Calculator {
    trait Service {
      def factorial(n: Int): UIO[BigInt]
    }
  }
  trait Calculator {
    def calculator: Calculator.Service
  }
  trait CalculatorLive extends Calculator {
    def calculator: Calculator.Service = new Calculator.Service {
      val numFibers = 10

      def factorial(n: Int): UIO[BigInt] =
        forkFibers(n, numFibers).foldLeft(UIO.succeed(BigInt(1)))(
          (totalIO, fiber) => totalIO.zipWith(fiber >>= (_.join))(_ * _))

      private def rangeFactorial(n: Long,
                                 lowerBound: Long,
                                 total: BigInt): BigInt =
        if (n <= lowerBound)
          total
        else
          rangeFactorial(n - 1, lowerBound, n * total)

      private def forkFibers(n: Int,
                             numFibers: Int): Seq[UIO[Fiber[Nothing, BigInt]]] =
        partialProductRanges(n, numFibers).map {
          case (h, l) => UIO(rangeFactorial(h, l, 1)).fork
        }

      def partialProductRanges(n: Int, fibers: Int): Seq[(Int, Int)] = {
        val numFibers = math.min(n, fibers)
        val step = scala.util.Try(n / numFibers).getOrElse(1)
        val remainder = scala.util.Try(n % numFibers).getOrElse(0)
        val genRange: Int => (Int, Int) = i => (i * step + step, i * step)
        if (remainder != 0)
          (0 until numFibers - 1)
            .map(genRange) :+ (n, n - remainder - step)
        else
          (0 until numFibers).map(genRange)
      }
    }
  }
  object CalculatorLive extends CalculatorLive

  object Validation {
    def validateInt(num: String): Option[Int] = scala.util.Try(num.toInt).toOption
  }

  val programZIO: ZIO[ConsoleIO with Calculator, Throwable, Unit] = for {
    _ <- consoleIO.putStr("Please enter a number: ")
    input <- consoleIO.getStr
    _ <- Validation
      .validateInt(input)
      .fold[ZIO[ConsoleIO with Calculator, Throwable, Unit]](
        consoleIO.putStr("Invalid number"))(number =>
        ZIO
          .accessM[Calculator](_.calculator.factorial(number)) >>= (factorial =>
          consoleIO.putStr(s"Factorial of $number is $factorial")))
  } yield ()

  object EnvironmentLive extends ConsoleIOLive with CalculatorLive

  val program = programZIO.provide(EnvironmentLive)

  val runtime = new DefaultRuntime {}

  runtime.unsafeRun(program)
}
```

## Fabio Labella

```scala
object Main extends App {
  // Too busy helping people on Gitter and Reddit
}
```

## Akka Scala programmer

Hates type checker as it slows down development. Has a huge short term memory capacity to keep track of all those messages.

```scala
import akka.actor.{Actor, ActorRef, ActorSystem, Props}

object Main extends App {
  object Console {
    case class Print(msg: String)
    case object GetLine
    case class UserInput(input: String)
  }
  class ConsoleActor extends Actor {
    def receive: Receive = {
      case Console.Print(msg) => print(msg)
      case Console.GetLine => {
        val input = scala.io.StdIn.readLine()
        sender ! Console.UserInput(input)
      }
    }
  }

  object Validation {
    case class ValidateInt(str: String)
    case class ValidInt(i: Int)
    case object InvalidValue
  }
  class ValidationActor extends Actor {
    def receive: Receive = {
      case Validation.ValidateInt(n) =>
        try {
          sender ! Validation.ValidInt(n.toInt)
        } catch {
          case _: Exception => sender ! Validation.InvalidValue
        }
    }
  }

  object Calculation {
    case class CalculateFactorial(n: Int)
    case class FactorialEquals(n: Int, result: BigInt)
  }
  class CalculationActor extends Actor {
    var factorial: ActorRef =
      system.actorOf(Props[FactorialActor], name = "factorial")
    var parent: ActorRef = _
    var n: Int = _

    def receive: Receive = {
      case Calculation.CalculateFactorial(number) =>
        n = number
        parent = sender
        factorial ! Factorial.Of(n)
      case Factorial.Done(result) =>
        parent ! Calculation.FactorialEquals(n, result)
    }
  }

  object Factorial {
    case class Of(n: Int)
    case class Next(n: Int, total: BigInt)
    case class Done(result: BigInt)
  }
  class FactorialActor extends Actor {
    import context._
    var parent: ActorRef = _

    def recurse: Receive = {
      case Factorial.Next(n, total) =>
        if (n <= 1) sender ! Factorial.Done(total)
        else sender ! Factorial.Next(n - 1, total * n)
      case Factorial.Done(result) =>
        parent ! Factorial.Done(result)
    }

    def receive: Receive = {
      case Factorial.Of(n) =>
        parent = sender
        become(recurse)
        self ! Factorial.Next(n, 1)
    }
  }

  object Program {
    case class Run(consoleActor: ActorRef,
                   validationActor: ActorRef,
                   calculationActor: ActorRef)
  }
  class ProgramActor extends Actor {
    var console: ActorRef = _
    var validation: ActorRef = _
    var calculation: ActorRef = _

    def receive: Receive = {
      case Console.UserInput(input) =>
        validation ! Validation.ValidateInt(input)
      case Program.Run(consl, vld, calc) =>
        console = consl
        validation = vld
        calculation = calc

        console ! Console.Print("Please enter a number: ")
        console ! Console.GetLine
      case Validation.ValidInt(n) =>
        calculation ! Calculation.CalculateFactorial(n)
      case Validation.InvalidValue =>
        console ! Console.Print("Invalid number")
        context.system.terminate()
      case Calculation.FactorialEquals(n, result) =>
        console ! Console.Print(s"Factorial of $n is $result")
        context.system.terminate()
    }
  }

  val system = ActorSystem("FactorialSystem")

  val consoleActor = system.actorOf(Props[ConsoleActor], name = "console")
  val validationActor =
    system.actorOf(Props[ValidationActor], name = "validation")
  val calculationActor =
    system.actorOf(Props[CalculationActor], name = "calculation")
  val programActor = system.actorOf(Props[ProgramActor], name = "program")

  programActor ! Program.Run(consoleActor, validationActor, calculationActor)
}
```

## Spark Scala programmer

Strongly believes that cannons are perfectly fine for killing flies. Doesn’t mess around.

```scala
import org.apache.spark.{SparkConf, SparkContext}

import scala.util.{Failure, Success, Try}

object Main extends App {

  def createSparkContext: SparkContext = {
    val conf = new SparkConf().setMaster("local[*]").setAppName("Factorial")
    val sc = new SparkContext(conf)
    sc.setLogLevel("ERROR")
    sc
  }

  def factorial(sparkContext: SparkContext, num: BigInt): BigInt =
    if (num == 0) BigInt(1)
    else {
      val list = (BigInt(1) to num).toList
      sparkContext.parallelize(list).reduce(_ * _)
    }

  print("Please enter a number: ")
  Try(scala.io.StdIn.readLine().toInt) match {
    case Success(n) =>
      val context = createSparkContext
      val fact = factorial(context, n)
      println(s"Factorial of $n is $fact")
    case Failure(_) =>
      println("Invalid number")
  }
}
```

## Java Scala programmer

Occasionally sneaks in Java class into a Scala project.

```scala
import java.io.BufferedReader;
import java.io.InputStreamReader;
import java.math.BigInteger;

public class Main {
    public static void main(String[] args) {
        System.out.print("Please enter a number: ");

        InputStreamReader read = new InputStreamReader(System.in);
        BufferedReader in = new BufferedReader(read);
        int number;

        try {
          number = Integer.parseInt(in.readLine());
          System.out.printf("Factorial of %d is %d", number, factorial(number));
        } catch (Exception e) {
            System.out.print("Invalid number");
        }
    }

    public static BigInteger factorial(int n) {
        BigInteger result = BigInteger.ONE;
        for (int i = 1; i <= n; i++) {
            result = result.multiply(BigInteger.valueOf(i));
        }
        return result;
    }
}
```

## Trampolined Scala programmer

Had a mental breakdown after that StackOverflowError happened in production on Friday evening. Hasn’t recovered to this day.

```scala
object Main extends App {

  sealed trait Trampoline[+A] {
    def map[B](f: A => B): Trampoline[B] = flatMap(f.andThen(Done(_)))
    def flatMap[B](f: A => Trampoline[B]): Trampoline[B] = FlatMap(this, f)
  }

  final case class Done[A](a: A) extends Trampoline[A]
  final case class More[A](resume: () => Trampoline[A]) extends Trampoline[A]
  final case class FlatMap[A, B](sub: Trampoline[A], k: A => Trampoline[B])
      extends Trampoline[B]

  def run[A](trampoline: Trampoline[A]): A = trampoline match {
    case Done(a) => a
    case More(r) => run(r())
    case FlatMap(sub, cont) =>
      sub match {
        case Done(a)       => run(cont(a))
        case More(r)       => run(FlatMap(r(), cont))
        case FlatMap(sub2, cont2) => run(sub2.flatMap(cont2(_).flatMap(cont)))
      }
  }

  def factorial(n: Int): Trampoline[BigInt] =
    if (n == 0) Done(1)
    else More(() => factorial(n - 1)).flatMap(x => Done(n * x))

  val program = FlatMap(
    Done(print("Please enter a number: ")),
    (_: Unit) =>
      FlatMap(
        Done(scala.util.Try(scala.io.StdIn.readLine().toInt).toOption),
        (maybeNumber: Option[Int]) =>
          maybeNumber.fold(Done(println("Invalid number")): Trampoline[Unit])(
            n => factorial(n).map(fact => println(s"Factorial of $n is $fact")))
    )
  )

  run(program)
}
```

## Peano ADT Scala programmer

Had a bright future but started hanging out with some bad Idris kids. Stopped caring about effects composition or stack safety.

```scala
object Main extends App {

  sealed trait Nat
  case object Z extends Nat
  case class S(n: Nat) extends Nat

  def natToInt(nat: Nat): Int = nat match {
    case Z    => 0
    case S(n) => 1 + natToInt(n)
  }

  def intToNat(i: Int): Nat = i match {
    case 0 => Z
    case n => S(intToNat(n - 1))
  }

  def plus(a: Nat, b: Nat): Nat = a match {
    case Z    => b
    case S(n) => S(plus(n, b))
  }

  def mult(a: Nat, b: Nat): Nat = a match {
    case Z    => Z
    case S(n) => plus(b, mult(n, b))
  }

  def factorial(n: Nat): Nat = n match {
    case Z    => S(Z)
    case S(m) => mult(n, factorial(m))
  }

  print("Please enter a number: ")
  scala.util.Try(scala.io.StdIn.readLine().toInt).toOption
    .fold(println("Invalid number"))(n =>
      println(s"Factorial of $n is ${natToInt(factorial(intToNat(n)))}"))
}
```

## Type level Scala programmer

Spends nights staring at those red squiggly lines, experiences genuine excitement when they disappear. Suffers from massive migraines and bad sleep quality. Favorite nonfiction — the HoTT book.

```scala
object Main extends App {

  trait Nat {
    type +[A <: Nat] <: Nat
    type *[A <: Nat] <: Nat
    type ! <: Nat
  }

  trait Z extends Nat {
    type +[A <: Nat] = A
    type *[A <: Nat] = Z
    type ! = _1
  }

  trait S[N <: Nat] extends Nat {
    type +[A <: Nat] = N# +[S[A]]
    type *[A <: Nat] = A# +[N# *[A]]
    type ! = S[N]# *[N# !]
  }

  type _1 = S[Z]
  type _2 = S[_1]
  type _3 = S[_2]
  type _4 = S[_3]
  type _5 = S[_4]
  type _6 = S[_5]

  implicitly[_3# ! =:= _6]
}
```

## Shapeless Scala programmer

Wears a beard and round glasses.

```scala
object Main extends App {
  import shapeless._
  import nat._
  import ops.nat._

  trait Factorial[I <: Nat] { type Out <: Nat }

  object Factorial {
    def factorial[N <: Nat](i: Nat)(implicit fact: Factorial.Aux[i.N, N],
                                    wn: Witness.Aux[N]): N = wn.value

    type Aux[I <: Nat, Out0 <: Nat] = Factorial[I] { type Out = Out0 }

    implicit def fact0: Aux[_0, _1] = new Factorial[_0] { type Out = _1 }
    implicit def factN[N <: Nat, F <: Nat, F1 <: Nat](
        implicit f: Factorial.Aux[N, F1],
        t: Prod.Aux[Succ[N], F1, F]): Aux[Succ[N], F] =
      new Factorial[Succ[N]] { type Out = F }
  }

  val program = for {
    _ <- Lazy(print("Please enter a number: "))
    _ <- Lazy(scala.io.StdIn.readLine())
    _ <- Lazy(println(
      s"Well what's more important is that a factorial of 5 is ${toInt(Factorial
        .factorial(5))} and it was calculated at compile time, how cool is that?"))
  } yield ()

  program.value
}
```

## Fix Scala programmer

Has a Y-combinator tattoo on his arm. Can stare at [romanesco broccoli](https://en.wikipedia.org/wiki/Romanesco_broccoli) for hours.

```scala
object Main extends App {

  def fix[A](f: (=> A) => A): A = {
    lazy val a: A = f(a)
    a
  }

  val gac: (=> BigInt => BigInt) => BigInt => BigInt =
    fac => n => if (n == 0) 1 else n * fac.apply(n - 1)

  val facF =
    fix[BigInt => BigInt](fac => n => if (n == 0) 1 else n * fac.apply(n - 1))

  val factorial = fix(gac)

  println(factorial(1000))
}
```

## Paramorphism Scala programmer

Actually a Haskell programmer in disguise. Uses vim with an empty `.vimrc`. Likes to draw commutative diagrams while listening to odd time signature progressive metal.

```scala
object Main extends App {

  trait Functor[F[_]] {
    def map[A, B](fa: F[A])(f: A => B): F[B]
  }

  sealed trait Nat[A]
  final case class Z[A]() extends Nat[A]
  final case class S[A](a: A) extends Nat[A]

  object Nat {
    implicit val natFunctor: Functor[Nat] = new Functor[Nat] {
      override def map[A, B](na: Nat[A])(f: A => B): Nat[B] =
        na match {
          case Z()  => Z()
          case S(a) => S(f(a))
        }
    }
  }

  final case class Fix[F[_]](unfix: F[Fix[F]])

  def ana[F[_], A](f: A => F[A])(a: A)(implicit F: Functor[F]): Fix[F] =
    Fix(F.map(f(a))(ana(f)))

  def cata[F[_], A](alg: F[A] => A)(e: Fix[F])(implicit F: Functor[F]): A =
    alg(F.map(e.unfix)(cata(alg)))

  def para[F[_], A](f: F[(Fix[F], A)] => A)(fix: Fix[F])(
      implicit F: Functor[F]): A =
    f(F.map(fix.unfix)(fix => (fix, para(f)(fix))))

  val natAlgebra: Nat[BigInt] => BigInt = {
    case Z()  => 1
    case S(n) => n + 1
  }

  val natAlgebraPara: Nat[(Fix[Nat], BigInt)] => BigInt = {
    case Z()           => 1
    case S((fix, acc)) => cata(natAlgebra)(fix) * acc
  }

  val natCoalgebra: BigInt => Nat[BigInt] =
    n => if (n == 0) Z() else S(n - 1)

  val factorial = (ana(natCoalgebra) _).andThen(para(natAlgebraPara))

  println(factorial(1000))
}
```

## Apo-futu-hylomorphism Scala programmer

Has lots of hair. Wrote this on a napkin in a bar while having some beer with Bartosz Milewski and Edward Kmett.

```scala
object Main extends App {

  trait Functor[F[_]] {
    def map[A, B](fa: F[A])(f: A => B): F[B]
  }

  final case class Fix[F[_]](unfix: F[Fix[F]])
  final case class Cofree[F[_], A](head: A, tail: F[Cofree[F, A]])
  sealed trait Free[F[_], A]
  final case class Continue[F[_], A](a: A) extends Free[F, A]
  final case class Combine[F[_], A](fa: F[Free[F, A]]) extends Free[F, A]

  object Free {
    def continue[F[_], A](a: A): Free[F, A] = Continue(a)
    def combine[F[_], A](fa: F[Free[F, A]]): Free[F, A] = Combine(fa)
  }

  def ana[F[_], A](f: A => F[A])(a: A)(implicit F: Functor[F]): Fix[F] =
    Fix(F.map(f(a))(ana(f)))

  def cata[F[_], A](alg: F[A] => A)(e: Fix[F])(implicit F: Functor[F]): A =
    alg(F.map(e.unfix)(cata(alg)))

  def hylo[F[_], A, B](f: F[B] => B)(g: A => F[A])(a: A)(
      implicit F: Functor[F]): B =
    f(F.map(g(a))(hylo(f)(g)))

  def apo[F[_], A](f: A => F[Either[Fix[F], A]])(
      implicit F: Functor[F]): A => Fix[F] =
    a =>
      Fix(F.map(f(a)) {
        case Left(fix) => fix
        case Right(aa) => apo(f).apply(aa)
      })

  def futu[F[_], A](f: A => F[Free[F, A]])(
      implicit F: Functor[F]): A => Fix[F] = {
    def toFix: Free[F, A] => Fix[F] = {
      case Continue(a) => futu(f).apply(a)
      case Combine(fa) => Fix(F.map(fa)(toFix))
    }

    a =>
      Fix(F.map(f(a))(toFix))
  }

  sealed trait Stack[A]
  final case class Done[A](result: BigInt) extends Stack[A]
  final case class More[A](a: A, next: BigInt) extends Stack[A]

  object Stack {
    implicit val stackFunctor: Functor[Stack] = new Functor[Stack] {
      override def map[A, B](sa: Stack[A])(f: A => B): Stack[B] =
        sa match {
          case Done(result)  => Done(result)
          case More(a, next) => More(f(a), next)
        }
    }
  }

  object HyloFactorial {
    val stackCoalgebra: BigInt => Stack[BigInt] =
      n => if (n > 0) More(n - 1, n) else Done(1)

    val stackAlgebra: Stack[BigInt] => BigInt = {
      case Done(result)    => result
      case More(acc, next) => acc * next
    }

    def factorial: BigInt => BigInt = hylo(stackAlgebra)(stackCoalgebra)
  }

  object FutuFactorial {
    val F: Functor[Stack] = implicitly[Functor[Stack]]

    val stackCoalgebraFutu: BigInt => Stack[Free[Stack, BigInt]] =
      n =>
        if (n > 0)
          F.map(More(n - 1, n))(Free.continue)
        else F.map(Done(1))(Free.continue)

    def factorial: BigInt => BigInt =
      (futu(stackCoalgebraFutu) _).andThen(cata(HyloFactorial.stackAlgebra))
  }

  object ApoFactorial {
    val stackCoalgebraApo: BigInt => Stack[Either[Fix[Stack], BigInt]] =
      n => if (n > 0) More(Right(n - 1), n) else Done(1)

    def factorial: BigInt => BigInt =
      (apo(stackCoalgebraApo) _).andThen(cata(HyloFactorial.stackAlgebra))
  }

  println(HyloFactorial.factorial(1000))
  println(FutuFactorial.factorial(1000))
  println(ApoFactorial.factorial(1000))
}
```

--- 

All code examples are [available on GitHub](https://github.com/vvviiimmm/scala-evolution)

### References:

Free monad 0: <https://typelevel.org/cats/datatypes/freemonad.html>
Free monad 1: <http://degoes.net/articles/modern-fp>
FreeK monad: <https://github.com/ProjectSeptemberInc/freek>
Recursion schemes: <https://free.cofree.io/2017/11/13/recursion>
Fix combinator: <https://free.cofree.io/2017/08/28/fixpoint>
Tagless final 0: <http://okmij.org/ftp/tagless-final/course/lecture.pdf>
Tagless final 1: <https://typelevel.org/blog/2018/06/27/optimizing-tagless-final-2.html>
ZIO (John A De Goes): <https://skillsmatter.com/skillscasts/13247-scala-matters>
ZIO: <https://zio.dev>
Shapeless: <https://underscore.io/books/shapeless-guide>
Type level computations: <http://slick.lightbend.com/talks/scalaio2014/Type-Level_Computations.pdf>
Peano numbers: <https://wiki.haskell.org/Peano_numbers>
Trampoline 0: <https://free.cofree.io/2017/08/24/trampoline>
Trampoline 1: <https://medium.com/@olxc/trampolining-and-stack-safety-in-scala-d8e86474ddfa>
Apache Spark: <https://spark.apache.org/docs/0.9.1/scala-programming-guide.html>
Akka: <https://doc.akka.io/docs/akka/current/index-actors.html?language=scala>