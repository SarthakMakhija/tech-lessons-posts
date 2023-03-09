---
author: "Sarthak Makhija"
title: "Kotlin DSL"
date: 2018-05-27
description: "A domain-specific language (DSL) is a computer language specialized to a particular application domain. This is in contrast to a general-purpose language (GPL), which is broadly applicable across domains. There are a wide variety of DSLs, ranging from widely used languages for common domains, such as HTML for web pages, down to languages used by only one or a few pieces of software. Let's explore DSL in Kotlin together."
tags: ["Domain specific language", "DSL", "Kotlin"]
thumbnail: /kotlin-dsl.png
caption: "Background by Alessio Soggetti on Unsplash"
---

> > A domain-specific language (DSL) is a computer language specialized to a particular application domain. This is in contrast to a general-purpose language (GPL), which is broadly applicable across domains. There are a wide variety of DSLs, ranging from widely used languages for common domains, such as HTML for web pages, down to languages used by only one or a few pieces of software.

### Kotlin DSL
Kotlin provides first class support for DSL which allows us to express domain-specific operations much more concisely than an equivalent piece of code in a general-purpose language.
Let's try and build a simple DSL in Kotlin -

```gradle 
dependencies {
   compile("io.arrow-kt:arrow-data:0.7.1")
   compile("io.arrow-kt:arrow-instances-core:0.7.1")
   testCompile("io.kotlintest:kotlintest-runner-junit5:3.1.0")
}
```
This should be familiar to people using <em>gradle</em> as their build tool. Above DSL specifies compile and testCompile dependencies for a gradle project in very concise and expressive form.

### How does Kotlin support DSL
Before we get in to Kotlin's support for DSL, let's look at lambdas in Kotlin.
```kotlin
fun buildString(action: (StringBuilder) -> Unit): String {
   val sb = StringBuilder()
   action(sb)
   return sb.toString()
}
```
`buildString()` takes a lambda as a parameter (called `action`) and invokes it by passing an instance of `StringBuilder`. Any client code which invokes `buildString()` will look like the below code -

```kotlin
val str = buildString {
    it.append("Hello")
    it.append(" ")
    it.append("World")
}
```
A few things to note here:
- `buildString()` takes lambda as the last parameter. If a function takes lambda as the last parameter, Kotlin allows you to invoke the function using braces { .. }, no need of using parentheses
- `it` is the implicit parameter available in lambda body which is an instance of `StringBuilder` in this example

This information is good enough to write a `gradle` dependencies DSL.

### First Attempt at DSL
In order to build a `gradle` dependencies DSL we need a function called `dependencies` which should take a lambda of type `T` as a parameter where `T` provides `compile` and `testCompile` functions.
Let's try:

```kotlin
fun dependencies(action: (DependencyHandler) -> Unit): DependencyHandler {
    val dependencies = DependencyHandler()
    action(dependencies)
    return dependencies
}

class DependencyHandler {
    fun compile(coordinate: String){
        //add coordinate to some collection
    }
    fun testCompile(coordinate: String){
        //add coordinate to some collection
    }
}
```
`dependencies` is a simple function which takes a lambda accepting an instance of `DependencyHandler` as a parameter and returns Unit. `DependencyHandler` is the type `T` which has `compile` and `testCompile` methods.
Client code for the above concept will look like -

```gradle
dependencies {
    it.compile("") //it is an instance of DependencyHandler
    it.testCompile("")
}
```
Are we done? Not really. The problem is the implicit parameter `it` is used in the client code. Can we remove `it`? To remove the implicit parameter, we need to look at "Lambda With Receiver".

### Lambda with receiver
Receiver is a simple type in Kotlin which is extended.
Let's see this with an example -

```kotlin
fun String.lastChar() : Char =
                   this.toCharArray().get(this.length - 1)
```

We have extended the `String` type to have `lastChar()` as a function which means we can always invoke it as -
```kotlin 
    "Kotlin".lastChar()
```
Here, `String` is the receiver type and `this` used in the body of the `lastChar()` function is the receiver object. Can we combine these 2 concepts - lambda and receiver?

Let's rewrite our `buildString` function using lambda with receiver -

```kotlin
fun buildString(action: StringBuilder.() -> Unit): String {
    val sb = StringBuilder()
    sb.action()
    return sb.toString()
}
```

A few things to note here:
- `buildString()` takes a lambda with receiver as a parameter
- `StringBuilder` is the receiver type in the lambda
- the way we invoke action function is different this time. 
 
> Because `action` is an extension function on `StringBuilder` we invoke it using `sb.action()`, where `sb` is an instance of `StringBuilder`

Let's now create a client of the `buildString` function -

```kotlin
val str = buildString {
    this.append("Hello") //this here is an instance of StringBuilder
    append(" ")
    append("World")
}
```
Isn't this brilliant? Client code will always have access to `this` while invoking a function which takes lambda with receiver as a parameter.
Shall we rewrite our `gradle` dependencies DSL code?

### Another Attempt at DSL

```kotlin
fun dependencies(action: DependencyHandler.() -> Unit): DependencyHandler {
    val dependencies = DependencyHandler()
    dependencies.action()
    return dependencies
}

class DependencyHandler {
    fun compile(coordinate: String){
        //add coordinate to some collection
    }
    fun testCompile(coordinate: String){
        //add coordinate to some collection
    }
}
```

The only change we have done here is in the `dependencies` function which takes a lambda with receiver as the parameter. `DependencyHandler` is the receiver type in the `action` parameter which means the client code will always have access to the instance of `DependencyHandler`.
Let's see the client code -

```kotlin
dependencies {
    compile("") //same as this.compile("")
    testCompile("")
}
```
We are able to create a DSL using lambda with receiver as a parameter to a function.

### Operator Function invoke()

Kotlin provides an interesting function called `invoke` which is an operator function. Specifying the `invoke` operator on a class allows it to be called on any instances of the class without a method name.
Let's see this in action:

```kotlin 
class Greeter(val greeting: String) {
    operator fun invoke(name: String) {
        println("$greeting $name")
    }
}

fun main(args: Array<String>) {
    val greeter = Greeter(greeting = "Welcome")
    greeter(name = "Kotlin")  //this calls the invoke function which takes String as a parameter
}
```

A few things to note about `invoke()` here:
- is an operator function
- takes a parameter
- can be overloaded
- is being called on the instance of the `Greeter` class without a method name

Let's use the `invoke` function in building the DSL.

### Building DSL using the invoke function

```kotlin 
class DependencyHandler {
    fun compile(coordinate: String){
        //add coordinate to some collection
    }
    fun testCompile(coordinate: String){
        //add coordinate to some collection
    }
    operator fun invoke(action: DependencyHandler.() -> Unit): DependencyHandler {
        this.action()
        return this
    }
}
```

We have defined an `operator function` in the `DependencyHandler` which takes a lambda with receiver as a parameter. This means `invoke` will automatically be called on instance(s) of `DependencyHandler` and the client code will have access to the instance of `DependencyHandler`.

Let's write the client code:

```kotlin 
val dependencies = DependencyHandler()
dependencies { //as good as dependencies.invoke(..)
   compile("")
   testCompile("")
}
```
The operator function `invoke()` can come in handy while building DSL.

### Conclusion

- Kotlin provides a first class support for DSL which is type safe
- One can create a DSL in Kotlin using -
  - Lambda as function parameters
  - Lambda with receiver as function parameter
  - Operator function `invoke` along with lambda with receiver as function parameter
  
### References
- Kotlin In Action