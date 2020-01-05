---
title: "Type classes explained"
classes: wide
categories:
  - FP
tags:
  - type classes
  - functional programming
  - scala
---

![image-center]({{ site.url }}{{ site.baseurl }}/assets/images/posts/typeclasses/header.jpeg){: .align-center}

Polymorphism is probably the most important feature in high level languages. It allows us to build programs according to interfaces, operate on abstractions and choose the right implementation based on types. Different languages implement it differently. Most OOP languages usually use inheritance and some kind of run time type dispatch or table lookup to get the right implementation. There is another way, which originally comes from Haskell which involves “**type classes**”. In this article we’ll go through the process of transforming traditional OOP style program into equivalent program with type classes in order to understand where the idea comes from.

## Polymorphism via inheritance
Let’s start with a classic OOP example — two dimensional shapes representation. We have a notion of a shape that has some area and multiple implementations for each kind of shape:

```scala
// Our generic inteface
trait Shape {
  def area: Double
}

// Implementation 1
class Circle(radius: Double) extends Shape {
  override def area: Double = math.Pi * math.pow(radius, 2)
}

// Implementation 2
class Rectangle(width: Double, length: Double) extends Shape {
  override def area: Double = width * length
}

// Generic function
def areaOf(shape: Shape): Double = shape.area

// Usage
areaOf(new Circle(10))
areaOf(new Rectangle(5, 5))
```

Take a look at how we use our custom implementations (last 2 lines). We’re creating an _instance of a class_ (which also contains implementation) and explicitly passing it as a parameter to our generic function which works on `Shape`s. Now let’s take that example and try to do the same using type classes. We will go through a number of transformations making small changes to our program to eventually come up with the solution. Not every step will make sense but I will try to explain the thought process behind it.

## Polymorphism via type classes

> OOP approach consolidates data and related functions in one place— class definition. “Type classes” go with a different approach — entities representing data are decoupled from entities responsible for implementation.
{: .notice}

It will sound strange and confusing at first. But no worries, it will make sense shortly.

As a first step, we will not directly extend our `Circle` and `Rectangle` from `Shape` but introduce a new class to do that:

```scala
trait Shape {
  def area: Double
}

// Shape definition data structures, should be in a diffrent file/namespace
case class Circle(radius: Double)
case class Rectangle(width: Double, length: Double)

// Implementation 1
class CircleShape(radius: Double) extends Shape {
  override def area = math.Pi * math.pow(radius, 2)
}

// Implementation 2
class RectangleShape(width: Double, length: Double) extends Shape {
  override def area: Double = width * length
}

def areaOf(shape: Shape): Double = shape.area

areaOf(new CircleShape(10))
areaOf(new RectangleShape(5, 5))
```

We have our `Circle` and `Rectangle` case classes while the fact that they can have an area is represented with `CircleShape` and `RectangleShape`. This is important, this is where the idea of separating data from functionality comes from.
Now, it doesn’t seem like we achieved anything, even worse, we have two problems now:

1. Code duplication — if we want to change `Circle` we would also need to change `CircleShape`.
1. To call `areaOf` we need to pass instance of `CircleShape`, not the `Circle` itself.

Lets try to fix the first problem:

```scala
trait Shape {
  def area: Double
}

case class Circle(radius: Double)
case class Rectangle(width: Double, length: Double)

//               v  (no constructor parameters)
class CircleShape extends Shape {
  override def area : Double = ??? // radius?
}

//                  v (no constructor parameters)
class RectangleShape extends Shape {
  override def area: Double = ??? // width and height?
}

def areaOf(shape: Shape): Double = shape.area

areaOf(new CircleShape)
areaOf(new RectangleShape)
```

Now `CircleShape` and `RectangleShape` don’t have any constructor parameters, thus no code duplication, great. The problem is — we needed them to calculate the area. `CircleShape` has to have information about `Circle`, but not in the constructor parameters. It feels like our area function is missing something. Imagine if instead of taking no arguments it would take a data structure representing the shape — `Circle` or `Rectangle` case classes:

```scala
// Won't compile
trait Shape {
  //       vvv (hmm)
  def area(???): Double
}

case class Circle(radius: Double)
case class Rectangle(width: Double, length: Double)

class CircleShape extends Shape {
  override def area(circle: Circle) : Double = math.Pi * math.pow(circle.radius, 2)
}

class RectangleShape extends Shape {
  override def area(rectangle: Rectangle): Double = rectangle.width * rectangle.length
}

//                                            vvv (but what about this?)
def areaOf(shape: Shape): Double = shape.area(???)
```

Good, now we have information to calculate the area. But how do you actually call `area`? And how should `area` declaration look like? We want the argument to be of type `Circle` or `Rectangle` but they don’t share any common ancestor (which was the point of separating them). Turns out we can do it easily, just have to introduce a type parameter:

```scala
// Shape is now a parametrized trait
// 'A' is placeholder for `Circle`, `Rectangle` or other shape representation
trait Shape[A] {
  def area(a: A): Double
}

case class Circle(radius: Double)
case class Rectangle(width: Double, length: Double)

// We have to extend with Shape[Circle] because we have to pass 'Circle' to `area`
class CircleShape extends Shape[Circle] {
  override def area(circle: Circle) : Double = math.Pi * math.pow(circle.radius, 2)
}

// Same here, 'area' takes 'Rectangle'
class RectangleShape extends Shape[Rectangle] {
  override def area(rectangle: Rectangle): Double = rectangle.width * rectangle.length
}

//                                                  vvv (hmmm)              
def areaOf[A](shape: Shape[A]): Double = shape.area(???)

areaOf(new CircleShape)
```

The key here is to extend parameterized `Shape` with the type we want our area to get the argument of. But it’s still not clear how to implement `areaOf` and thread actual `Circle` into `shape.area()`. Well, we don’t even pass any information about the shape we want to get the area for. Clearly we need to change `areaOf` and add second parameter with actual shape info:

```scala
//            vvvvvvvvvvvv (new parameter)
def areaOf[A](shapeInfo: A, shape: Shape[A]): Double = shape.area(shapeInfo)

areaOf(Circle(10), new CircleShape)
```

Cool, but let me just rename the arguments to make more sense:

```scala
def areaOf[A](shape: A, shapeImpl: Shape[A]): Double = shapeImpl.area(shape)

areaOf(Circle(10), new CircleShape)
```

`shape` is our shape definition (`Circle` or `Rectangle`) while `shapeImpl` is an implementation of a `Shape` interface for that particular definition.

Its all good but we actually cheated a little bit by changing our `areaOf` function. The calls from the very first OOP example looked like `areaOf(new Circle(10))`. Now we have to pass some weird implementation object as an additional parameter. We know that we do need that implementation object but we don’t want to pass it explicitly. Is that possible?

The answer is yes, moreover, scala has this as a language feature called **implicit parameters**. (Check out this this article explaining implicits)

Lets modify our `areaOf` function, `shapeImpl` will be passed as an implicit parameter:

```scala
//                      vvvvvvvv
def areaOf[A](shape: A)(implicit shapeImpl: Shape[A]): Double = shapeImpl.area(shape)

// No more implementation objects
areaOf(Circle(10))
```

`areaOf` takes a single parameter now but the code won’t compile. We need implicit variables to be in scope:

```scala
implicit val circleShape = new CircleShape
implicit val rectangleShape = new RectangleShape

areaOf(Circle(10))
areaOf(Rectangle(5, 5))
```

It feels like a boilerplate, we have to instantiate them or import to the scope each time we want to call `areaOf`. Can we instantiate them only once and not think about having them in scope? Yes, **implicit objects** to the rescue:

```scala
trait Shape[A] {
  def area(a: A): Double
}

case class Circle(radius: Double)
case class Rectangle(width: Double, length: Double)

// Here >
implicit object CircleShape extends Shape[Circle] {
  override def area(circle: Circle) : Double = math.Pi * math.pow(circle.radius, 2)
}

// And here >
implicit object RectangleShape extends Shape[Rectangle] {
  override def area(rectangle: Rectangle): Double = rectangle.width * rectangle.length
}

def areaOf[A](shape: A)(implicit shapeImpl: Shape[A]): Double = shapeImpl.area(shape)

areaOf(Circle(10))
areaOf(Rectangle(5,5))
```

That’s it! Implementation objects are singletons instantiated once and available for the compiler.
Hope it all made sense and if not — no worries, the concept is not easy to understand at first. Get your hands on, create your own type classes and check out the references to understand the concept better.

--- 

#### References

1. Cordelia Hall, Kevin Hammond, Simon Peyton Jones, and Philip Wadler, "Type classes in Haskell" <https://homepages.inf.ed.ac.uk/wadler/papers/classhask/classhask.ps>
2. Bryan O'Sullivan, Don Stewart, John Goerzen, "Chapter 6. Using Typeclasses" in Real World Haskell <http://book.realworldhaskell.org/read/using-typeclasses.html>
3. Learn You a Haskell: Types and Typeclasses <http://learnyouahaskell.com/types-and-typeclasses>
4. A Gentle Introduction to Haskell, Version 98: Type Classes and Overloading <https://www.haskell.org/tutorial/classes.html>