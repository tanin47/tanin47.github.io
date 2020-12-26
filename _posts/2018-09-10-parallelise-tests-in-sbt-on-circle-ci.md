---
layout: post
title: "Parallelize tests in SBT on CircleCI"
description: "Make your CI faster on CircleCI by parallelizing it with this SBT trick."
date: 2018-09-10
category: technical
---

In my previous post on [parallelising tests in SBT](https://tanin.nanakorn.com/technical/2018/07/04/parallelise-tests-in-sbt.html), it doesn't work well in practice or, at least, on CircleCI. The main disadvantage is that it doesn't balance the tests by their run time. Balancing tests by their run time would reduce the total time significantly.

CircleCI offers the command-line tool, named [circleci](https://circleci.com/docs/2.0/parallelism-faster-jobs/), obviously, for splitting tests. One mode of splitting is splitting tests based on how long individual tests take. If `scala_test_classnames` contains a list of test classes, we can split using `circleci tests split --split-by=timings --timings-type=classname scala_test_classnames`. The output is a list of test classes that should be run according to the machine numbered `CIRCLE_NODE_INDEX` out of the `CIRCLE_NODE_TOTAL` machines. `circleci` conveniently reads these two env variables automatically.

At a high level, we want sbt to print out all the test classes. We feed those classes to `circleci`. Then, we feed the output of `circleci` to `sbt testOnly`. Finally, we use CircleCI's `store_test_results` to [store the test results which includes time](https://circleci.com/docs/2.0/parallelism-faster-jobs/#splitting-by-timings-data). `circleci` uses this info to split tests accordingly in subsequential runs.

Now it's time to write an SBT task again. This task is straigtforward because, when googling "sbt list all tests", the answer is [one of the first items](https://stackoverflow.com/questions/20332802/how-to-get-a-list-of-defined-tests-in-sbt-project-in-sbt-0-12). Here's how I do it in my codebase:

```
val printTests = taskKey[Unit]("Print full class names of tests to the file `test-full-class-names.log`.")

printTests := {
  import java.io._

  println("Print full class names of tests to the file `test-full-class-names.log`.")

  val pw = new PrintWriter(new File("test-full-class-names.log" ))
  (definedTests in Test).value.sortBy(_.name).foreach { t =>
    pw.println(t.name)
  }
  pw.close()
}
```

Then, in `.circleci/config.yml`, we can use the below commands:

```
...
  - run: sbt printTests
  - run: sbt "testOnly  $(circleci tests split --split-by=timings --timings-type=classname test-full-class-names.log | tr '\n' ' ') -- -u ./test-results/junit"
...
```

Please notice that:

* `testOnly` takes a list of class names delimited by space. But `circleci` outputs a list of class names delimited by `\n`. Therefore, we fix this using `tr`.
* `-u` is for writing test results in junit-style xml to the specified directory, `./test-results/junit`.

Finally, we need to `store_test_results`. It looks like this in our `.circleci/config.yml`:

```
...
  - store_test_results:
      path: ./test-results
...
```

Please note that `store_test_results` requires the xml file to be in a subdirectory of `./test-results` ([See reference](https://circleci.com/docs/2.0/configuration-reference/#store_test_results)).

And there you go! Now your SBT tests are parallelised on CircleCI with sensible balancing.
