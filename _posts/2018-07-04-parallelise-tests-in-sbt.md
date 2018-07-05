---
layout: post
title: "Parallelize tests in SBT with frustration"
date: 2018-07-04
category: technical
---

<a href="https://www.scala-sbt.org/">SBT</a>, the official build tool for Scala, is a very complex build tool. It's one of those things that makes me wonder if I am stupid or the tool's complexity surpasses average human intelligence.

I've done a few things with SBT (e.g. printing the list of all tests), usually using the trial-and-error approach, and this time I want to add test parallelism to my project.

The requirement is straightforward. Heroku CI or CircleCI can run our tests on multiple machines. In each machine, 2 environment variables, say, `MACHINE_INDEX` and `MACHINE_NUM_TOTAL`, are set. We can use these 2 environment variables to shard our tests.

<!---excerpt--->

Conceptually, the solution is simple: we read the list of all tests, decide which tests to run based on the 2 environment variables, and feed this information to an underlying code of `sbt "testOnly <test_class_path1> <test_class_path2>"`.

My first version that doesn't work was:

```scala
val shardedTests = taskKey[Unit]("Run sharded tests according to MACHINE_INDEX and MACHINE_NUM_TOTAL")

shardedTests := {
  println("Hello") // This is printed after `(Test /testOnly).value`
  def allFalse(s: String) = {
    println("invoked"); // this is not printed
    false // This is a dummy implementation to test the integration first.
  }
  Test / testOnly / testFilter := {
    println("The setting is executed") // This is not printed
    { ss => ss.map { s => allFalse(_) } }
  }
  (Test / testOnly).value
}
```

Surprisingly, this didn't work. All tests were run. When I added `println`s in `allFalse` and in `testFilter`, nothing was printed. And when I added `println("Hello")` at the beginning of `shardedTests`, `(Test / testOnly).value` was still executed first.

This was when I've learned that an SBT task is actually the root of a dependency tree. When we refers to another task in the body, the referred task is magically executed first regardless of the order of the statements.

And, strangely enough, `testFilter`'s type is of type `Seq[String] => Seq[String => Boolean]`.

After a lot of trials, I still couldn't figure out how to do it until I stumbled on a solution where it suggested me to copy the whole configuration of `Test` and set `testOptions`. Here's how it looks like:

```scala
lazy val root = (project in file("."))
  .configs(shardedTest)
  .settings(inConfig(shardedTest)(Defaults.testTasks):_*)

lazy val shardedTest = config("shardedTest") extend Test

testOptions in shardedTest := Seq(Tests.Filter { s => (s.hashCode % sys.env("MACHINE_INDEX").toInt) == sys.env("MACHINE_NUM_TOTAL").toInt }) // We can later make the sharding algorithm smarter.
```

Running `sbt shardedTest:test` seems to shard tests correctly.

Now I've learned that we can't just set keys within a task's body and expect the task to pick it up. We have to make a new scope (which is config) in order to override certain settings.

Admittedly, I still don't fully understand what we did here, especially this part `.settings(inConfig(shardedTests)(Defaults.testTasks):_*)`. I merely know it makes a copy of something.

There are 4 points shown here why SBT is so counter-intuitive to me:
1. The order of statements within a task body doesn't seem to matter much when a task depends on another task.
2. We can set a key in a task body, but the setting won't really do anything. We should have just made it fail compilation.
3. There doesn't seem to be a good way to debug why a test command doesn't pick up the setting.
4. I can't seem to understand SBT's source code. For example, I have spent some time on [this part of code](https://github.com/sbt/sbt/blob/1.x/main/src/main/scala/sbt/Defaults.scala#L706) to see which setting I should overwrite, but I couldn't understand it.

