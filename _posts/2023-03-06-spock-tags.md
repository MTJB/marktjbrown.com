---
layout: post
title:  Splitting tests using JUnit Tags
description: Grouping together tests can help you to parallelize your integration tests and make your build times fas...
date:   2023-03-06 15:01:35 +0300
image:  '/images/posts/2023-03-06-spock-tags/title.png'
tags:   [java, groovy, spock]
---

## Summary 📖
As a project scales, there will (should) be more tests and in a perfect world, these new tests should not have a noticeable
change to the build times of your application. But the world is not perfect, developers are lazy, and after a short time
your build times have ballooned 🎈

Common ideology in software circles is the concept of the
[testing pyramid](https://martinfowler.com/articles/practical-test-pyramid.html), but if you find yourself working on a
project that has an inverted testing pyramid (that is, a greater proportion of integration-style testing) - you probably
have slow build pipelines. A colleague once referred to this style of testing as the testing "funeral urn" ⚱️, which is
rather fitting to how you might feel waiting over 2 hours to get build results. 🥲

It's not feasible to re-implement tens of thousands of tests overnight, so grouping together
tests can help you to parallelize your integration tests and make your build times faster
(you're gonna need a bigger build box).

## Who might find this useful? 🤔
If any of the below apply to you - you might be in the right place. Don't worry, I won't judge you - nor will I tell you
how many apply to my projects...

1. Have you got a slow build?
2. Have you got a lot of slow tests?
3. Are you using [Spock](https://spockframework.org/) and/or [Junit 5](https://junit.org/junit5/)?
4. Are you currently using JUnit [categories](https://junit.org/junit4/javadoc/4.12/org/junit/experimental/categories/Categories.html) to group your tests, but are trying to upgrade to newer flavours of Spock and/or JUnit?
5. Are you trying to get to Java 17 which is causing your to also upgrade Gradle, Groovy, and Spock and are you losing your mind and regretting how many tests you have written in Spock?

Ok fine, they all apply to me.

## Tag, you're it 🏷️
JUnit 5 added [tag](https://junit.org/junit5/docs/current/user-guide/#writing-tests-tagging-and-filtering) support - which
provides a way to categorize tests and later filter them based on some tag expressions. Tags are a full replacement of
[categories](https://junit.org/junit4/javadoc/4.12/org/junit/experimental/categories/Categories.html), which required a lot
of boilerplate code to implement - with tags, an annotation does the trick;

```java
import org.junit.jupiter.api.Assertions;
import org.junit.jupiter.api.Tag;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.CsvSource;

@Tag("JUnit")
public class DivideJunitTests {

    @Tag("Divide")
    @ParameterizedTest
    @CsvSource({"10,10,1", "10,1,10", "4200,10,420"})
    public void testDivide(int a, int b, int expected) {
        Assertions.assertEquals(expected, Divide.divide(a, b));
    }
}
```

The idea then is that you can scatter these tags across your test suites - and speed up your builds by running each of
the tags in parallel 💨

And the good news for those who are using a hybrid cocktail Spock _and_ JUnit is that Spock uses a JUnit
runner under the hood. [Spock 2.3](https://spockframework.org/spock/docs/2.3/release_notes.html#_release_notes) introduced
its own tags which play nicely with the JUnit variant.

## Code 🧑‍💻
All the code in this post can be found on my [GitHub](https://github.com/MTJB/example-junit-spock-tags) 🖖

_Disclaimer: it's generally not advised to duplicate all of your tests in both Java and Groovy like these examples -
that most likely leads to less than optimal build time, too._

Suppose you have 2 test classes - one in JUnit;
```java
import org.junit.jupiter.api.Assertions;
import org.junit.jupiter.api.Tag;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.CsvSource;

@Tag("JUnit")
public class DivideJunitTests {

    @Tag("Divide")
    @ParameterizedTest
    @CsvSource({"10,10,1", "10,1,10", "4200,10,420"})
    public void testDivide(int a, int b, int expected) {
        Assertions.assertEquals(expected, Divide.divide(a, b));
    }
}
```

..and one in Groovy;
```groovy
import spock.lang.Specification
import spock.lang.Tag

@Tag("Spock")
class DivideSpockTests extends Specification {

    @Tag("Divide")
    void "Test divide"(int a, int b, int expected) {
        expect:
            Divide.divide(a, b) == expected
        where:
            a    | b  | expected
            10   | 10 | 1
            10   | 1  | 10
            4200 | 10 | 420
    }
}
```

As expected, these can both be run via Gradle;
```
$ ./gradlew test

> Task :test

DivideSpockTests > Test divide > com.mtjb.demo.math.DivideSpockTests.Test divide [a: 10, b: 10, expected: 1, #0] PASSED

DivideSpockTests > Test divide > com.mtjb.demo.math.DivideSpockTests.Test divide [a: 10, b: 1, expected: 10, #1] PASSED

DivideSpockTests > Test divide > com.mtjb.demo.math.DivideSpockTests.Test divide [a: 4200, b: 10, expected: 420, #2] PASSED

DivideJunitTests > testDivide(int, int, int) > com.mtjb.demo.math.DivideJunitTests.testDivide(int, int, int)[1] PASSED

DivideJunitTests > testDivide(int, int, int) > com.mtjb.demo.math.DivideJunitTests.testDivide(int, int, int)[2] PASSED

DivideJunitTests > testDivide(int, int, int) > com.mtjb.demo.math.DivideJunitTests.testDivide(int, int, int)[3] PASSED

BUILD SUCCESSFUL in 3s
5 actionable tasks: 5 executed
```


With a few tweaks to your build file;
```groovy
test {
    useJUnitPlatform {

        def groups = System.getProperty("groups")
        println(groups)
        if (groups != null) {
            includeTags(groups)
        }
    }
    testLogging {
        events "passed", "skipped", "failed"
    }
}
```

You can start to filter what tests are being ran;
```
$ ./gradlew test -Dgroups=Spock

> Task :test

DivideSpockTests > Test divide > com.mtjb.demo.math.DivideSpockTests.Test divide [a: 10, b: 10, expected: 1, #0] PASSED

DivideSpockTests > Test divide > com.mtjb.demo.math.DivideSpockTests.Test divide [a: 10, b: 1, expected: 10, #1] PASSED

DivideSpockTests > Test divide > com.mtjb.demo.math.DivideSpockTests.Test divide [a: 4200, b: 10, expected: 420, #2] PASSED

BUILD SUCCESSFUL in 3s
5 actionable tasks: 5 executed
```

## ⚡️ Conclusion
JUnit tags, unsurprisingly, works as expected! If you're like me, and are working on a project with a web of dependencies
you might find this harder than it ought to be - and for that I advise some self-reflection to see if you can figure out
why you've written such confusing build files (or if you're like me, to curse that guy who used to work here!).