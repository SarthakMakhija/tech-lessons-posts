---
author: "Sarthak Makhija"
title: "Kotlin Wishlist for Java"
date: 2018-04-20
description: "There is no doubt that Java has enjoyed a superior position when it comes to programming languages and is considered as one of the most important languages for development. However, there have been a number of languages developed on top of the JVM like Kotlin. After working on the \"data-anonymization\" project I realized that there are things that Java should consider importing from Kotlin."
tags: ["Kotlin", "Java"]
thumbnail: /wishlist.jpg
caption: "Moved from dzone"
---

There is no doubt that Java has enjoyed a superior position when it comes to programming languages and is considered as one of the most important languages for development. However, there have been a number of languages developed on top of the JVM, like <a href="https://kotlinlang.org/" target="_blank" rel="nofollow noopener noreferrer">Kotlin</a>
Kotlin is a statically typed programming language for modern multi-platform applications. While I have been a Java developer for quite a long while, working on the <a href="https://github.com/dataanon/data-anon" target="_blank" rel="nofollow noopener noreferrer">data-anonymization</a> project made me feel that there are things that Java should consider importing from Kotlin.
These are some Kotlin features that I would love to see making a place in Java.

### Promote Immutability
Java 9 promotes immutability by introducing factory methods to create collections. It would be great to see immutable collections embedded in the language, rather than relying on wrappers to generate immutable collections. `existingDepartments()` is a function that returns an immutable list of `Strings` in Kotlin.
```kotlin
fun existingDepartments(): List =
    listOf("Human Assets", "Learning & Development", "Research")
```
Java 9 comes closest to returning an immutable list by throwing an `UnsupportedOperationException` when an attempt is made to add or remove an element from the list. It would be great to have a separate hierarchy of mutable and immutable collections and avoid exposing add/remove or any other mutating methods from immutable collections.

```java
//pre Java 8
public List existingDepartments() {
    return new ArrayList(){{
        add("Human Assets");
        add("Learning & Development");
        add("Research");
    }};
}
```

```java
//Java 8
public List existingDepartments() {
    return Stream.of("Human Assets", "Learning & Development", "Research")
                 .collect(Collectors.toList());
}
```

```java
//Java 9
public List existingDepartments() {
    return List.of("Human Assets", 
                   "Learning & Development",
                   "Research");
}
```
> Being more explicit about immutable collections and letting immutable collections speak out loud for themselves should be given preference over exposing methods and throwing `UnsupportedOperationExceptions`.

### Method Parameters Should Be Final by Default
With an intent to promote immutability and avoid errors because of mutation, it might be worth to at least giving a thought to making method parameters final by default.
```Kotlin
fun add (augend: Int, addend: Int) = augend + addend
```

Parameters for the `add()` function are `val` by default cannot be changed, which means as a client of any function, I can rest assured that the function is not changing the arguments (*not to be confused with object mutation*) that are passed to it.
> Making method parameters final by default might and will most likely break existing code bases on Java upgrades but is worth giving a thought.

### Handle NULL at Compile Time
Every Java developer is bound to know the infamous `NullPointerException`. Kotlin took a major step by handling NULLs at compile time. Everything is non-null be default unless it is explicitly stated.
Did Java 8 not introduce `Optional` for the very same reason ? Let's see with an example:

```Kotlin
class Employee(private val id: Int, private val department: Department?) {
    fun departmentName() = department?.name ?: "Unassigned"
}
class Department(val name: String)
/**
Employee needs a non-nullable "id" and an optional department to be constructed.
val employee = Employee(null, null); //Compile Time Error
 **/
```

The `Employee` class has a primary constructor with a non-nullable `id` and an optional (nullable) `department`. Passing null for the `id` will result in a compilation error.
The `departmentName` function accesses the name property of `Department` using the optional operator `?` on the nullable field. If `department` is null, `name` will not be accessed and the expression on the left-hand side `department?.name` will return null. The `Elvis` operator `?:` will return the right-hand side ("Unassigned") if the left-hand side of the expression is null.

```java
//Java8
class Employee {
    private Integer id;
    private Optional department;

    Employee(Integer id, Optional department){
       this.id = id;
       this.department = department;
    }
    public String departmentName() {
       return department.orElse("Unassigned");
    }
}
/**
Employee needs a non-nullable "id" and an optional department to be constructed.
Employee employee = new Employee(null, null); //NPE;
**/
```

Optional will not protect the code from NPE, but Optional has its advantages:

- It makes the domain model clear. The `Employee` class has an `optional department`, which is good enough to conclude that every employee may not be assigned a department
- It promotes composability as in the case of the `departmentName` method

> Handling NULLs at compile time should result in cleaner code by removing unnecessary NULL checks in the form of an if statement, `Objects.requireNonNull`, `Preconditions.checkNotNull`, any other form.

### Improve Lambdas
Java 8 introduced lambdas, which are built on top of a functional interface and a functional descriptor, meaning every lambda expression will map to the signature of an abstract method defined in that functional interface. This means it is a mandate to have an interface (Functional Interface) with only one abstract method (Functional Descriptor) to create a lambda expression.
```kotlin
val isPositive: (Int) -> Boolean = { it > 0 }
OR,
val isPositive: (Int) -> Boolean = { num > 0 }
OR,
val isPositive: (Int) -> Boolean = { num: Int > 0 }

//Usage
isPositive(10) returns true
isPositive(-1) returns false
```
In the above code, the variable `isPositive` a function that takes an `Int` as an argument and returns a `Boolean`. The value of this variable is a function definition or a lambda defined in braces, which checks that the passed argument is greater than zero.
Whereas, as seen in Java below, `Predicate` is a functional interface containing an abstract method `test()` — which takes an argument of type `T` and returns a `boolean`.
So, `isPositive` takes an argument of type `Integer` and checks that it is greater than zero. In order to use it, we need to invoke the `test()` method on the `isPositive` method.

```java
//Java 8
private Predicate<Integer> isPositive = (Integer arg) -> arg > 0;

//Usage
isPositive.test(10) returns true
isPositive.test(-1) returns false

@FunctionalInterface
public interface Predicate<T> {
    boolean test(T t);
}
```
> Lambdas should be independent of functional interfaces and their functional descriptors.

### Support Extension Functions
Kotlin supports extension functions, which provide the ability to extend a class with new functionality without having to inherit from the class or use any type of design pattern, such as Decorator.
Let's write an extension function to return the last character of a String, meaning `"Kotin".lastChar()` will return 'n'.

```kotlin
fun String.lastChar() = this.toCharArray()[this.length - 1]
/**
    Extension functions are of the form -
    fun <ReceiverObject>.function_name() = body
    OR,
    fun <ReceiverObject>.function_name(arg1: Type1, ... argN: TypeN) = body
**/
```

Here, `lastChar()` is an extension function defined on the `String` type, which is called a receiver object. This function can now be invoked as `"Kotlin".lastChar()`.
> Extension functions provide an ability to extend a class with new functionalities without inheritance or any other design pattern.

### Tail Recursion
Kotlin provides support for <a href="https://kotlinlang.org/docs/reference/functions.html" target="_blank" rel="nofollow noopener noreferrer">Tail-recursion</a>. Tail-recursion is a form of recursion in which the recursive calls are the last instructions in the function (tail). In this way, we don't care about previous values, and one stack frame suffices for all the recursive calls; tail-recursion is one way of optimizing recursive algorithms.
The other advantage/optimization is that there is an easy way to transform a tail-recursive algorithm to an equivalent one that uses iteration instead of recursion.

```kotlin
fun factorialTco(val: Int): Int {
    tailrec fun factorial(n: Int, acc: Int): Int = if ( n == 1 ) acc else factorial(n - 1, acc * n)
  return  factorial(val, acc = 1)
}
```
When a function is marked with the `tailrec` modifier and meets the required form, the compiler optimizes out the recursion, leaving behind a fast and efficient loop-based version instead.
> Effectively, a tail-recursive function can execute in constant stack space, so it's really just another formulation of an iterative process

Java does not directly support tail-call optimization at the compiler level, but one can use <a href="http://blog.agiledeveloper.com/2013/01/functional-programming-in-java-is-quite.html" target="_blank" rel="nofollow noopener noreferrer">lambda expressions</a> to implement it. It would be nice to see TCO at the compiler level.

### Miscellaneous

**Remove inherent duplication [new, return, semicolon]:** Kotlin does not require `new` to create an instance. It still needs a `return` if a function is treated as a statement instead of an expression.

```kotlin
class Employee(private val id: Int, private val department: Department?) {
    //no return
    fun departmentNameWithoutReturn() = department?.name ?: "Unassigned"

    //return is needed if a function is treated as a statement rather than an expression
    fun departmentNameWithoutReturn() {
        val departmentName = department?.name ?: "Unassigned"
        return departmentName
    }
}
```

**Singleton Classes**: It would be great to see an easier way to create singleton classes in Java. An equivalent syntax in Kotlin is seen below.</li>

```kotlin
object DataProviderManager {
    fun registerDataProvider(provider: DataProvider) {
        // ...
    }
}
```

**Immutable Classes**: It would be good to see something like a `readonly` or `immutable` modifier to create an immutable class. The below-mentioned code snippet is simply a thought (not available in Kotlin or Java).

```java
//Hypothetical
immutable class User(private val name: String, private val id: Int)
```

### Conclusion
As developers, we will always make mistakes (skipping NULL checks, mutating a collection, etc.), but providing features at the language level that can stop these mistakes will make our lives easier and prevent mistakes.
