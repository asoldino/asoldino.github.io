## A practical walkthrough to Monads

### Why effects?
Not every possible problem can be described in terms of pure functions, and let's admit it, without the notorious side effects it is not even possible to write meaningful computer programs.

### Errors

Let's start from a very simple example:

```scala
def divide(num: Int, den: Int): Int = num / den
```

Whilst it is pretty obvious what this function does, can it really be evaluated for _every_ combination of num and den? Quite unsurprisingly, not: in fact what happens when den is _0_?

```scala
val plusInfinite = divide(42, 0)

java.lang.ArithmeticException: / by zero
at .divide(<console>:12)
... 38 elided
```

Even admitting there is a way for an Int to host the mathematical equivalent of infinite, a `java.lang.ArithmeticException` is quite outside the codomain of _Int_, isn't it? Something is not quite right, but living with exceptions is quite normal for every programmer.

So, is this already the end of the functional programming dream? Is even a simple arithmetic division not possible? Of course not, but it is necessary to rethink about the codomain a little. From mathematics, we know that it is not possible (for natural numbers) to divide by zero: this means that the codomain must be aware of that. This is when effects come into play!

A natural way of modelling such a situation is the following:

```scala
sealed trait MaybeInt
case class JustA(n: Int) extends MaybeInt
case class Nothing extends MaybeInt
```

Following this reasoning, we just modeled a type that can disjointly express the presence of a number (JustA(42)) or Nothing, in case an error occurred (we tried to divide by zero). Turns out that in many languages it is possible to add template parameters to type, such that the same effect can be applied to integers, strings, DatabaseRowBeans, et. al.

In scala, this class of effects is called Option[+A] (for every type A):

```scala
sealed abstract class Option[+A] extends Product with Serializable {...}
final case class Some[+A] {...}
case object None extends Option[Nothing] {...}

val someInt: Option[Int] = Some(42)
val noInt: Option[Int] = None
```

With this new tool at our disposal, we can now rewrite divide:

```scala
def betterDivide(num: Int, den: Int): Option[Int] = (num, den) match {
  case (_, 0)          => None
  case (a, b)          => Some(a / b)
}
```

Here, using a pattern match betterDivide is defined, such that any number divided by 0 returns None, otherwise Some(result). The full domain is now covered by the codomain.

```scala
betterDivide(1, 2)
res4: Option[Int] = Some(0)

betterDivide(2, 1)
res5: Option[Int] = Some(2)

betterDivide(2, 0)
res6: Option[Int] = None
```

Wrapping the result into an effect was necessary to retain a predictable result in case of errors. It is not without price though, for example this code would not compile:

```scala
val result = betterDivide(42, betterDivide(25, 2))

Error:(29, 38) type mismatch;
found   : Option[Int]
required: Int
betterDivide(2, betterDivide(2, 2))
```

It is necessary to "unwrap" the value from the effect first. Option offers an API named `get` to the rescue:

```scala
betterDivide(2, betterDivide(2, 2).get)

res7: Option[Int] = Some(2)
```

But we are back to square one in case the Option is not yielding a value (i.e. is describing an error situation):

```scala
betterDivide(2, betterDivide(2, 0).get)

java.util.NoSuchElementException: None.get
```

### Values available after a non-fixed time

To model computations that may yield a result, or fail, in a later moment, scala developers naturally use Futures. Future is another effect useful to separate the concern of keeping track of an activity (potentially executed in a different thread from the caller) from the pure functional concern of manipulating the result.

As an example, a fictional database holds the response to the question about the meaning of life, the universe, everything. Of course, to obtain it, one must know the question first! In case the question is correct, then the database returns "42!", otherwise _nothing_. To reuse the same well-known concepts, an Option[String] will be used as codomain. To model the latency that is usually necessary to invoke an external database, a Future[Option[String]] will wrap the result of the invocation.

```scala
def resultOfTheCallToTheDatabase(secretKey: String): Option[String] =
  if (secretKey == "the meaning of life, the universe, everything") Some("42!")
  else None

def realCallToTheDatabase(secretKey: String): Future[Option[String]] =
  Future {
    Thread.sleep(500) // Easy way to simulate a bit of latency

    resultOfTheCallToTheDatabase(secretKey)
  }
```

Invoking `realCallTotheDatabase` right away will not work, not even compile actually.

```scala
realCallToTheDatabase("forgot!")

Error:(49, 16) Cannot find an implicit ExecutionContext. You might pass
an (implicit ec: ExecutionContext) parameter to your method
or import scala.concurrent.ExecutionContext.Implicits.global.
Future {
```

The compiler is looking for an ExecutionContext to use, in order to be able to execute the future asynchronously. The default ExecutionContext is available as an import:

```scala
import scala.concurrent.ExecutionContext.Implicits.global

realCallToTheDatabase("forgot!")

res9: scala.concurrent.Future[Option[String]] = Future(<not completed>)
```

Result is marked to be `<not completed>` yet. The thread sleep of half a second is quite nasty. Similarly to the Option[A], with the get, Future offers a way to unwrap a value from its context using the API `Await.result`

```scala
Await.result(realCallToTheDatabase("the meaning of life, the universe, everything"), Duration.Inf)

res10: Option[String] = Some(42!)
```

### Working with effects

`betterDivide` is not composable with itself, in the sense it is not possible to express something like `(4 / (2 / 2))` without first unwrapping the inner division result. Trying to change the domain changing `num` and `den` to Option[Int] is changing the domain itself, so it doesn't really solve the issue. Writing a more complex program by `reusing` betterDivide is possible, but requires a different approach. Option[A] has a method called `flatMap` that, by accepting a function from the wrapped A to an Option[B], returns the Option[B], for any B (also when B == A). The quirk is that if there is a value to compute, then the function is evaluated. In case the Option[A] is None, then None is returned immediately as Option[B], no extra computation required, no null checks, no `NullPointerExceptions`. This is called fail-fast computation.

```scala
@inline final def flatMap[B](f: A => Option[B]): Option[B] = {...}
```

The program can be written as:

```scala
betterDivide(2, 2).flatMap(den => betterDivide(4, den))

res11: Option[Int] = Some(4)

//which is equivalent to
for {
  den <- betterDivide(2, 2)

  result <- betterDivide(4, den)
} yield result

res12: Option[Int] = Some(4)

//unsurprisingly
for {
  num <- betterDivide(2, 0)

  den <- betterDivide(42, 2)

  result <- betterDivide(num, den)
} yield result

res13: Option[Int] = None
```

Future[A] has a similar `flatMap` method that, by accepting a function from an A to a Future[B], it returns the Future[B]. In the case of futures, it means another long-running asynchronous is started immediately after the first one, yielding an A, ends. Is there a pattern here?

```scala
for {
  theResponse <- realCallToTheDatabase("the meaning of life, the universe, everything")
  
  _ <- Future { 
    stareForFifteenSeconds() 
  
    println(s"I got the response finally, it is $theResponse") 
  }
} yield ()
```

### Can we talk about Monads finally?

Effects can be seen as a computation description (errors, latency, IO, ...); `flatMap` is like a glueing tool to chain an effect to another sequentially. It is valid for Options, and also for Futures. Sounds like a pattern.

> Monads in Haskell can be thought of as composable computation descriptions. The essence of monad is thus separation of composition timeline from the composed computation's execution timeline, as well as the ability of computation to implicitly carry extra data, as pertaining to the computation itself, in addition to its one (hence the name) output, that it will produce when run (or queried, or called upon). This lends monads to supplementing pure calculations with features like I/O, common environment, updatable state, etc.
> -The Haskell Wiki: https://wiki.haskell.org/Monad

The Haskell definition of Monad now probably makes a bit of sense. Computations descriptions come through effects, composition comes in term of invoking flatMap after flatMap, extra data is carried over in sense of Thread management, if necessary to run async operations. In Scala there are several libraries that model Monads, this post will use Cats.

> Each monad, or computation type, provides means, subject to Monad Laws, to
> (a) create a description of a computation that will produce (a.k.a. "return") a given Haskell value, and
> (b) combine (a.k.a. "bind") a computation description with a reaction to it, – a pure Haskell function that is set to receive a computation-produced value (when and if that happens) and return another computation description, using or dependent on that value if need be, – creating a description of a combined computation that will feed the original computation's output through the reaction while automatically taking care of the particulars of the computational process itself.
> -The Haskell Wiki: https://wiki.haskell.org/Monad

`bind` is exactly `flatMap`, whilst `return` is the concept of wrapping a pure value into an effect in a generic way, for example wrapping 42 into Some(42).

Example of Monad invocation using Cats is:

```scala
import cats.Monad
import cats.instances.option._

Monad[Option].pure(42)

res14: Option[Int] = Some(42)

Monad[Option].flatMap(betterDivide(2, 3))(num => betterDivide(num, 42))

res15: Option[Int] = Some(0)
```

or even something like (Option is not IO nor Writer but let's assume this is acceptable):

```scala
for {
  _ <- Monad[Option].pure(println("Starting..."))

  num <- betterDivide(2, 3)

  _ <- Monad[Option].pure(println(s"Numerator will be $num"))

  den <- betterDivide(42, 2)

  result <- betterDivide(num, den)
} yield result

Starting...
Numerator will be 0
res16: Option[Int] = Some(0)
```

### A powerful ally, it is!

```scala
def sum(a: Int, b: Int) = a + b

def sumF[F[_] : Monad](a: Int, b: Int): F[Int] = Monad[F].pure(sum(a, b))

sumF[Option](4, 2)

res19: Option[Int] = Some(6)

sumF[Future](4, 2)

res20: scala.concurrent.Future[Int] = Future(Success(6))

sumF[List](4, 2)

res21: List[Int] = List(6)

// Wait, List also has a monad?
Monad[List].flatMap(List(4,8))(x => List(x - 1, x, x + 1))

res22: List[Int] = List(3, 4, 5, 7, 8, 9)
```
