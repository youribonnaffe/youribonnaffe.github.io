---
layout: post
title:  "Using Java Records for real: Parameterized Unit tests"
date:   2021-01-02 12:00:00 +0100
categories: java records
---

[Records](https://openjdk.java.net/jeps/359) are a new feature of the Java language introduced in version 14. This is still a preview feature, meaning that the exact specification is pretty much defined but that feedback is still welcome before making it part of the JDK. The [--preview compiler flag](https://www.baeldung.com/java-preview-features) can be used to enable such features. While they are still a preview feature in Java 15, we can already start playing around with them and try to see how they will improve our lives as Java developers.

In this article we are showing how records can be used to implement parameterized unit tests using JUnit 5. In its version 5, JUnit introduces the [`@ParameterizedTest`](https://junit.org/junit5/docs/current/user-guide/#writing-tests-parameterized-tests) annotation along with [`@MethodSource`](https://junit.org/junit5/docs/current/user-guide/#writing-tests-parameterized-tests-sources-MethodSource). Here is an example on how it can be used:

{% highlight java %}
class ParameterizedTestCase {

    static Stream<Arguments> testData() {
        return Stream.of(
                Arguments.of("apple", 5),
                Arguments.of("banana", 6));
    }

    @ParameterizedTest
    @MethodSource("testData")
    void countLetters(String input, int output) {
        assertEquals(output, input.length());
    }
}
{% endhighlight %}

The `testData` method will be used to generate as many test executions as defined and the arguments will be passed to each execution.
Below is how the execution looks like in Intellij IDEA.
When executed, the test data are automatically displayed by JUnit 5 part of the test methods:

![Test execution without records](/assets/images/2021-01-02/no-records.png#spacing)

Now the [`Arguments`](https://junit.org/junit5/docs/5.2.0/api/org/junit/jupiter/params/provider/Arguments.html) interface is easy to use but maybe lacks readability or type checking we would expect from Java. We could introduce our own class instead of arguments. It would be a simple class with a few immutable fields and accessors. Just what records are! Let's refactor the test a bit:

{% highlight java %}
class JavaRecordsTestCase {

    record TestCase(String input, int output) {}

    static Stream<TestCase> testCases() {
        return Stream.of(
                new TestCase("apple", 5),
                new TestCase("banana", 6));
    }

    @ParameterizedTest
    @MethodSource("testCases")
    void countLetters(TestCase testCase) {
        assertEquals(testCase.output(), testCase.input().length());
    }
}
{% endhighlight %}

Now with just one line of code we can provide more meaning to our test data with types and names. The `TestCase` record effectively provides a short definition for an immutable object with accessors.

![Test execution without records](/assets//images/2021-01-02/records.png#spacing)

As records come with the [`toString`]() method out of the box, it even shows in the generated method names when tests are executed. Here the example is kept simple but you can easily imagine a real use case with more than a few inputs and outputs to check.
