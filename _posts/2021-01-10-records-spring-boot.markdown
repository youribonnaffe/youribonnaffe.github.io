---
layout: post
title:  "Using Java Records for real: Spring Boot"
date:   2021-01-10 12:00:00 +0100
categories: java records spring
---

Can [Java records](https://openjdk.java.net/jeps/359) already be used with [Spring Boot](https://spring.io/projects/spring-boot)? 
Thanks to Jackson's support of records, yes they can!

> If you are not yet familiar with Java records you might want to start by looking at the official proposal in [JEP 359](https://openjdk.java.net/jeps/359) or at [the official documentation](https://docs.oracle.com/en/java/javase/14/docs/api/java.base/java/lang/Record.html).

[Jackson](https://github.com/FasterXML/jackson) is a serialization/deserialization library supporting JSON (and many other formats). 
Spring Boot uses it to accept and return JSON content from the controllers for instance.

In version [2.12](https://cowtowncoder.medium.com/jackson-2-12-features-eee9456fec75), the support of Java records has been added to Jackson. 
It should be noted that the latest version of Spring Boot (2.4.1 as of this writing) is still using Jackson 2.11.x so you would
have to override the version in your build tool to use records.

Here is an example of a _build.gradle_ file to use records with Spring Boot:

{% highlight groovy %}
plugins {
	id 'java'
	id 'org.springframework.boot' version '2.4.1'
	// provide the dependencyManagement DSL
	id 'io.spring.dependency-management' version '1.0.10.RELEASE'
}

// We want to use the latest version of Java
sourceCompatibility = '15'

repositories {
	mavenCentral()
}

dependencyManagement {
	imports {
		// override with latest Jackson versions
		mavenBom 'com.fasterxml.jackson:jackson-bom:2.12.1'
	}
}

// Records are still a preview feature so we need to enable it
tasks.withType(JavaCompile) {
	options.compilerArgs += "--enable-preview"
}
tasks.withType(Test) {
	jvmArgs += "--enable-preview"
}
tasks.withType(JavaExec) {
	jvmArgs += "--enable-preview"
}
{% endhighlight %}

Now let's see how we can leverage records to simplify our day to day usage of Spring Boot.

# Using records for configuration

Spring Boot supports mapping configuration to beans using [`@ConfigurationProperties`](https://www.baeldung.com/configuration-properties-in-spring-boot).
You just need to provide a simple bean with getters and setters for it to be automatically mapped.
Jackson is the underlying library used to perform this mapping so we can leverage it to use records instead.

With the following configuration in the `application.yaml` file:

{% highlight yaml %}
custom:
  aProperty: 123
  anotherProperty: value
{% endhighlight %}

and the following record:

{% highlight java %}
@ConfigurationProperties(prefix = "custom")
@ConstructorBinding
public record RecordConfiguration(int aProperty, String anotherProperty) { }
{% endhighlight %}

We would be able to retrieve and inject an instance of `RecordConfiguration` with just this content: _RecordConfiguration[aProperty=123, anotherProperty=value]_.
 `@ConfigurationProperties` lets us map the properties starting with the _custom_ prefix.
 As we used a record that does not come with setters we added the [`@ConstructorBinding`](https://docs.spring.io/spring-boot/docs/current/api/org/springframework/boot/context/properties/ConstructorBinding.html) annotation to rely on the constructor instead.

 Configuration properties are just a perfect match for Java records, immutable objects with read-only properties.
 The default `toString()` implementation is useful for debugging purposes (although you might want to be careful 
 about logging sensitive configuration data).

# Using records in controllers

Another place in Spring Boot's applications where immutable objects would make sense is for controllers.
Most of the tutorials you will find out there will show you how you can take an object and pass it as a controller parameter 
down to be persisted in the database. This is convenient to demonstrate the simplicity of Spring Boot but we all know that
once our application starts to become a little more complex we will benefit from decoupling the different layers.

At the controller layer we usually need simple POJOs, just a representation of the data that is received and sent back over the wire.
Yet another good use case for records!

Let's take a look at the [RESTful Web Service guide from Spring](https://spring.io/guides/gs/rest-service/), here we define a simple controller that returns a `Greeting` object:

{% highlight java %}
@PostMapping("/greeting")
    public Greeting greeting(@RequestBody Greeting greeting) {
        return new Greeting(counter.incrementAndGet(), 
            String.format(template, greeting.content()));
}
{% endhighlight %}

With `Greeting` defined as:

{% highlight java %}
public class Greeting {

	private final long id;
	private final String content;

	public Greeting(long id, String content) {
		this.id = id;
		this.content = content;
	}

	public long getId() {
		return id;
	}

	public String getContent() {
		return content;
	}
}
{% endhighlight %}

🧐 This structure looks familiar...

{% highlight java %}
public record Greeting(long id, String content) { }
{% endhighlight %}

That's it, just as the guide says: 
_This application uses the Jackson JSON library to automatically marshal instances of type Greeting into JSON. Jackson is included by default by the web starter_. 

So we are indeed good to go just writing records to serialize data to JSON in our Spring web services.
It also works the same way to deserialize data so records can be used as input parameters for request bodies too.

# Using records with templates

Now maybe you are not only returning JSON content but also using [templates](https://spring.io/guides/gs/serving-web-content/). Another area where records can help!

Here is a simple template that displays data about books:

{% highlight html %}
<div th:object="${book}">
    <span th:text="*{author}"></span> wrote <span th:text="*{title}"></span>
</div>
{% endhighlight %}

And a controller using this template:

{% highlight java %}
@GetMapping("/template")
public String template(Model model) {
    
    record Book(String author, String title){}

    model.addAttribute("book", new Book("Orwell", "1984"));
    return "mytemplate";
}
{% endhighlight %}

Here we even created an inline record, just passing around the required data to the template.
However if you want to benefit from [Intellij support for Thymeleaf](https://www.jetbrains.com/help/idea/thymeleaf.html)
 you might want to make it a proper class to annotate its usage with `@thymesVar`.

# (Not) Using records to persist data

Could we use records to persist objects to a database?

The short answer is no, records are not well suited for persistence, mainly because Hibernates does not just save or read data from
the database but manages the entire state, potentially generating update statements. Something that would be hard to achieve with immutable
objects such as records.

This [blog post](https://vladmihalcea.com/java-records-jpa-hibernate/) by [Vlad Mihalcea](https://vladmihalcea.com/) explains it 
very well and offers one use case for records: DTO projections.

_Here we looked only at relational databases using Hibernate, with other databases, such as MongoDB the story might be different though._

# Conclusion

In this blog post we showed how Java records can be used within a Spring Boot application:
- to map configuration properties to objects
- to serialize and deserialize JSON data in controllers
- to pass data in templates

These are just a few places that came to mind where records can be a good fit, there are probably more so feel free to continue
the discussion on [Twitter](https://twitter.com/youribm)!