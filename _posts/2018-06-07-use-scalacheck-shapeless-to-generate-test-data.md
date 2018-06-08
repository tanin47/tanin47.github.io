---
layout: post
title: Use scalacheck-shapeless to generate test data
date: 2018-06-07 19:41:00 -08:00
---

At [GIVE.asia](https://give.asia), we are using Playframework and Slick. Slick uses case classes as models.

<!---excerpt--->

For example, here's the User model:

```scala
import com.twitter.util.Time

case class User(
  id: Long,
  name: String,
  email: String,
  createdAt: Time)
```

The verbosity problem arose when we needed `User` in tests. Since we didn't want to specify all parameters every time we instantiated `User`, we made a helper function like the below:

```scala
object Maker {
  def user(
    id: Long = 1L,
    name: String = "name",
    email: String = "email@give.asia",
    createdAt: Time = Time.now
  ) = User(id = id, name = name, email = email, createdAt = createdAt)
}
```

The helper alleviated the verbosity problem to some degree. If we only cared about `email`, we could do `Maker.user(email = "invalid-email")`. It worked for a while until we have so many models and fields. Our `Maker` contained a thousand line of code. That was when I started to look for a new solution.

Deep in my heart, I have always known that we can solve this verbosity problem with Scala Macros. I just didn't know how. I have been asking around some time ago: [here](https://www.reddit.com/r/scala/comments/7fgvjo/how_to_generate_a_function_to_instantiate_a_case/) and [here](https://stackoverflow.com/questions/47482542/generate-a-function-to-instantiate-a-case-class-with-default-values-using-scala).

One suggestion is around [scalacheck-shapeless](https://github.com/alexarchambault/scalacheck-shapeless). It turns out, with scalacheck-shapeless, we can instantiate a case class with `implicitly[Arbitrary[YourCaseClass]].arbitrary.sample.get`.

However, that line is still verbose. But, since scalacheck-shapeless has already done a lot heavy-lifting, we can extend it with Scala Macros to make it much less verbose. Here's the code:

```scala
// This object should be in a dependent project. Macros' code must be compiled before the main project's compilation.
object MakerMacro {
  def applyImpl[T: c.WeakTypeTag](c: whitebox.Context): c.Expr[T] = {

    import c.universe._

    c.Expr[T](
      q"""
        import org.scalacheck.{Arbitrary, Gen}
        import org.scalacheck.ScalacheckShapeless._
        import ${c.prefix}._ // This is to import the target scope, so that the line below can access `implicitTime`.
        implicitly[Arbitrary[${weakTypeTag[T].tpe}]].arbitrary.sample.get
       """
    )
  }
}

// The code below is in your main project.
import org.scalacheck.{Arbitrary, Gen}

object Maker {
  implicit val implicitTime = Arbitrary.apply(Gen.const(Time.now)) // We need this because shapeless doesn't know how to generate `Time`.

  def apply[T]: T = macro MakerMacro.applyImpl[T]
}

Maker[User].copy(email = "some-email") // Instantiate `User` with random values.
```

Now the code is very terse :)

Edit: there are some good suggestions in [this reddit thread](https://www.reddit.com/r/scala/comments/8kaguf/use_scalacheckshapeless_to_instantiate_a_case/)
