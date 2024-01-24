---
author: "Sarthak Makhija"
title: "Diving into Rust by building an assertions crate"
date: 2024-01-24
description: "
As Rust projects grow in size and complexity, the need for sophisticated error handling tools becomes ever more pressing. 
Traditional methods like panics and asserts, while useful, can be limited and cumbersome.
Let's build an assertions crate that offers elegant and powerful assertions, while simultaneously diving into the diverse landscape of Rust features. 
"
tags: ["Rust", "Assertions", "Elegant-assertions", "Clearcheck"]
thumbnail: /diving-into-rust.webp
caption: "Background by Dana Tentis on Pexels"
---

As Rust projects grow in size and complexity, the need for sophisticated error handling tools becomes ever more pressing. 
Traditional methods like panics and asserts, while useful, can be limited and cumbersome.

Let's build an assertions crate that offers elegant and powerful assertions, while simultaneously diving into the diverse landscape of Rust features. 

### Introduction

Let's define some requirements for our crate. The assertions crate should:

1. **Offer Fluent API**: chain assertions for a natural and readable experience.
2. **Have extensive assertions**: variety of assertions covering common validation needs.
3. **Be customizable**: extend with custom assertions for specific domain requirements.
4. **Be type-safe**: leverage Rust's type system for reliable assertions.

Consider this password validation example as a glimpse into the crate's ability to create meaningful and expressive assertions. 

```rust
let pass_phrase = "P@@sw0rd1 zebra alpha";
pass_phrase.should_not_be_empty()
    .should_have_at_least_length(10)
    .should_contain_all_characters(vec!['@', ' '])
    .should_contain_a_digit()
    .should_not_contain_ignoring_case("pass")
    .should_not_contain_ignoring_case("word");
```

### Understanding String, &str and str types

It is important to understand various string types in Rust before we get started.

- **String**: is a mutable and resizable buffer holding UTF-8 text. The buffer is allocated on heap. It can be treated as a collection of u8, `vec<u8>`.
- **&str**: is an immutable reference to a run of UTF-8 text owned by someone else. `&str` is a fat pointer, it contains both
  the address of the actual string and its length.
- **str**: is an immutable sequence of UTF-8 bytes of dynamic length somewhere in memory. It's size in unknown, which means `str` almost always 
appears as `&str` in the code.

The below code and the visual highlights the types: `String`, and `&str`.

```rust
let value: String = String::from("RUST");
let value_ref: &str = &value[1..];
let literal: &str = "TRUST";
```

<div class="align-center-exclude-width-change">
    <img src="/string-vs-str.webp" alt="String vs &str"/>
    <figcaption class="figcaption">Visual from the book: Programming Rust</figcaption>
</div>

> In Rust, the string literals are of type &str and the actual literals are allocated on pre-allocated read-only
> memory.

### Beginning with extension methods

The first thing that we need is the ability to invoke custom methods like `should_contain_all_characters` and `should_be_empty` on the built-in
types. In the password validation example, we have methods like `should_contain_all_characters` and `should_contain_a_digit` 
over `string` (or `&str`) types.

Such methods are called "extension" methods and [Wikipedia](https://en.wikipedia.org/wiki/Extension_method#:~:text=Extension%20methods%20are%20features%20of,an%20equally%20safe%20manner%2C%20however.) defines extension method as
"a method added to an object after the original object was compiled". **In Rust, extension methods are added using traits.**

Let's start by defining a trait.

```rust
pub trait MembershipAssertion {
    fn should_contain_a_digit(&self) -> &Self;
    fn should_not_contain_a_digit(&self) -> &Self;
    fn should_not_be_empty(&self) -> &Self;
}
```

`MembershipAssertion` defines methods: `should_contain_a_digit`, `should_not_contain_a_digit` and `should_not_be_empty`, all of which returns reference to `Self` to allow chaining.

Let's implement the trait for a string slice (`&str`). *(I will only implement should_contain_a_digit at this stage).*

```rust
impl MembershipAssertion for &str {
    fn should_contain_a_digit(&self) -> &Self {
        let contains_a_digit = self.chars().any(|ch| ch.is_numeric());
        if !contains_a_digit {
            panic!("assertion failed: {:?} should contain a digit", self);
        }
        self
    }
}
```

Time to add a few tests. *(I will only show a couple of tests.).*

```rust
#[cfg(test)]
mod tests {
    use crate::MembershipAssertion;

    #[test]
    fn should_contain_a_digit() {
        let password = "P@@sw0rd1 zebra alpha";
        password.should_contain_a_digit();
    }

    #[test]
    fn should_not_be_empty_and_contain_a_digit() {
        let password = "P@@sw0rd1 zebra alpha";
        password.should_not_be_empty()
                .should_contain_a_digit();
    }
}
```

It is a decent start, and we can make our first commit.

We have implemented the `MembershipAssertion` trait for the `&str` type. We also need the methods of the `MembershipAssertion` trait for the `String` type.

Let's implement the trait for the `String` type. 

```rust
impl MembershipAssertion for String {
    fn should_contain_a_digit(&self) -> &Self {
        let contains_a_digit = self.chars().any(|ch| ch.is_numeric());
        if !contains_a_digit {
            panic!("assertion failed: {:?} should contain a digit", self);
        }
        self
    }
}

#[cfg(test)]
mod string_tests {
    use crate::MembershipAssertion;

    #[test]
    fn should_contain_a_digit() {
        let password = String::from("P@@sw0rd1 zebra alpha");
        password.should_contain_a_digit();
    }

    #[test]
    fn should_not_be_empty_and_contain_a_digit() {
        let password = String::from("P@@sw0rd1 zebra alpha");
        password.should_not_be_empty().should_contain_a_digit();
    }
}
```

We have simply duplicated the method `should_contain_a_digit` for `&str` and `String` types and verified that duplicated code works for both the types. 

Let's make an attempt to remove the duplication.

We can convert the `String` type to a `string slice -> &str` and invoke the respective methods on `&str` type. With this approach, the implementation
of `MembershipAssertion` on `String` looks like the following:

```rust
impl MembershipAssertion for String {
    fn should_contain_a_digit(&self) -> &Self {
        (self as &str).should_contain_a_digit();
        self
    }

    fn should_not_be_empty(&self) -> &Self {
        (self as &str).should_not_be_empty();
        self
    }
}
```

This approach removes the duplication, but it is still unnecessary delegation - all the methods in the `String` implementation delegate 
to the respective implementation in the `&str` type. This might not look like a huge thing but if we add a few more methods in the `MembershipAssertion`
trait, the delegation will become obvious. So the question is, can we do better?

### Understanding AsRef

Let's revisit our `MembershipAssertion` trait.

```rust
pub trait MembershipAssertion {
    fn should_contain_a_digit(&self) -> &Self;
    fn should_not_contain_a_digit(&self) -> &Self;
    fn should_not_be_empty(&self) -> &Self;
}
```

The idea is to provide an implementation of `MembershipAssertion` for all the types which can be represented as string. 

It will be great to implement `MembershipAssertion` for any generic type `T`, where `T` that it can be represented as `String` (or `&str`).

Let's give this concept a try.

```rust
impl<T> MembershipAssertion for T
    where T: AsRef<str> + Debug {
    fn should_contain_a_digit(&self) -> &Self {
        let contains_a_digit = self.as_ref().chars().any(|ch| ch.is_numeric());
        if !contains_a_digit {
            panic!("assertion failed: {:?} should contain a digit", self);
        }
        self
    }
}
```

We are implementing `MembershipAssertion` for any `T` where T implements [AsRef<str>](https://doc.rust-lang.org/std/convert/trait.AsRef.html) and [Debug](https://doc.rust-lang.org/std/fmt/trait.Debug.html) trait.

Jim Blandy, Jason Orendorff and Leonora F.S Tindall in the book [Programming Rust](https://www.oreilly.com/library/view/programming-rust-2nd/9781492052586/) says: when a type implements `AsRef<U>`, it means we can borrow a `reference of U` from the type.
This means, if a type implements `AsRef<str>` we can borrow a `reference of str` from that type.

In Rust, both `String` and `&str` type implements `AsRef<str>` which means all the methods of `MembershipAssertion` are now available on `String`
and `&str`. Please find the reference [here](https://doc.rust-lang.org/std/convert/trait.AsRef.html#:~:text=Since%20both%20String%20and%20%26str,accept%20both%20as%20input%20argument). 

We need the `T` to implement `Debug` trait because we are formatting it in the `panic` macro.

Quick revisit: 
- Rust allows implementing a trait for any `T`. This is called *blanket trait implementation*. In our case, we are implementing `MembershipAssertion`
for a constrained `T`.
- The trait `AsRef` provides a lot of flexibility in the function signature. The method [open](https://doc.rust-lang.org/std/fs/struct.File.html#method.open) in
Rust [File](https://doc.rust-lang.org/std/fs/struct.File.html) type takes a parameter `P` of type `AsRef<Path>`, which means any `P` that can be represented as a [Path](https://doc.rust-lang.org/std/path/struct.Path.html)
can be passed as an argument.
```rust
// open method from Rust's file type.
pub fn open<P: AsRef<Path>>(path: P) -> Result<File> {..}
```

With the introduction of `AsRef<str>` change, we have taken care of the duplication in the implementation of `MembershipAssertion`, and we can
now remove our original implementation of `MembershipAssertion` for `String` and `&str` types.

### Leveraging blanket trait implementation

Rust allows implementing a trait for any generic type `T`. Consider a trait `BoxWrap` as shown below:

```rust
pub trait BoxWrap {
    fn boxed(self) -> Box<Self>;
}
```

This trait provides `boxed` method to create a [Box](https://doc.rust-lang.org/std/boxed/struct.Box.html) representation of self. *Please note
the boxed method takes the ownership of `self`*.

> In Rust, [Box](https://doc.rust-lang.org/std/boxed/struct.Box.html) is a pointer type that uniquely owns a heap allocation of type T.
> `Box::new(x: T)` allocates memory on heap and then places `x` into it.

We can use blanket trait implementation to implement the method `boxed` over any generic type `T`.

```rust
impl<T> BoxWrap for T {
    fn boxed(self) -> Box<Self> {
        Box::new(self)
    }
}
```

That's it, the method `boxed` is now available on all the types `T`.

```rust
fn main() {
    let value: &str = "clearcheck";
    println!("{:?}", value.boxed());

    let id: i32 = 120;
    println!("{:?}", id.boxed());
}
```

This concept is very useful for our crate. Let's see how. 

Our crate will also offer assertions related to ordered comparisons (greater than, less than, greater than equal to, less than equal to) for various types. Let's
provide such a trait.

```rust
pub trait OrderedAssertion<T: PartialOrd> {
    fn should_be_greater_than(&self, other: T) -> &Self;
    fn should_not_be_greater_than(&self, other: T) -> &Self;
    // other methods
}
```
The trait `OrderedAssertion` is generic over `T: PartialOrd`. [PartialOrd](https://doc.rust-lang.org/std/cmp/trait.PartialOrd.html) trait allows ordered comparisons (>, <, >=, <=) between types.

We have an option of implementing `OrderedAssertion` for various types like `i8`, `i16`, `i32`, `&str` etc. or we can implement `OrderedAssertion`
for a constrained `T`.

Let's make an attempt to implement `OrderedAssertion` for any type `T` that implements `PartialOrd`.

```rust
impl<T> OrderedAssertion<T> for T
    where T: PartialOrd + Debug {
    fn should_be_greater_than(&self, other: T) -> &Self {
        if self <= &other {
            panic!("{:?} should be greater than {:?}", self, other)
        }
        self
    }

    fn should_not_be_greater_than(&self, other: T) -> &Self {
        if self > &other {
            panic!("{:?} should not be greater than {:?}", self, other)
        }
        self
    }
}
```

The methods `should_be_greater_than` and `should_not_be_greater_than` are now available on any `T` which implements `PartialOrd` trait.

Note, the operators (>=, <) translate to a method call [partial_cmp](https://doc.rust-lang.org/std/cmp/trait.PartialOrd.html#tymethod.partial_cmp) inside the `PartialOrd` trait. 
The method `partial_cmp` takes parameters by reference for comparison. Hence, we are doing the comparisons using references `(self <= &other)`.

```rust
#[cfg(test)]
mod ordering_tests {
    use crate::OrderedAssertion;

    #[test]
    fn should_be_greater_than_other() {
        let rank = 12.90;
        rank.should_be_greater_than(10.23);
    }

    #[test]
    fn should_be_greater_than_other_with_string() {
        let name = "junit";
        name.should_be_greater_than("assert");
    }
}
```

The advantage of implementing a trait for any `T` (with or without constraint) is obvious. It removes duplication!!! 

Time to revisit the implementation of our assertion.

### Introducing Matchers

Let's take a look at the implementation of `OrderedAssertion` again.

```rust
impl<T> OrderedAssertion<T> for T
    where T: PartialOrd + Debug {
    fn should_be_greater_than(&self, other: T) -> &Self {
        if self <= &other {
            panic!("{:?} should be greater than {:?}", self, other)
        }
        self
    }

    fn should_not_be_greater_than(&self, other: T) -> &Self {
        if self > &other {
            panic!("{:?} should not be greater than {:?}", self, other)
        }
        self
    }
}
```

There are two issues with this implementation.

#### Subtle duplication

There is still subtle duplication between the implementations of `should_be_greater_than` and `should_not_be_greater_than`. 

Let's take a closer look to understand the duplication:
- In comparison: `self <= &other` and `self > &other`, both the methods are doing comparisons which are inverted.
- In the panic message: `panic!("{:?} should be greater than {:?}", self, other)` and `panic!("{:?} should not be greater than {:?}", self, other)`,
both the methods are causing panic with messages which are inverted.

The same form of duplication will be observed between method pairs like: 
- `should_be_empty`, `should_not_be_empty` 
- `should_be_greater_than`, `should_not_be_greater_than`
- etc

This duplication is caused because of the change in the condition for examining the data: *self must be **less than** other*, *self must be **greater than** other*.

#### The lack of ability to compose assertions

Assertions are defining the contract, ensuring that each data type (/data structure) adheres to its intended behavior. They don't provide us
with an ability to compose them using operators like **and**, **or**.

We can deal with both these issues by introducing an abstraction that:
- **Examines the data** and **verifies** that the data conforms to **specific criteria**.
- **Inverts** its behavior so that we **won't** be required to write any **additional code** for implementing **negative assertions**.
- Allows **composition**

The question is what would be the name of such an abstraction? In the world of assertions, such an object is called matcher. I asked [bard](https://bard.google.com/chat) to define *Matcher*? It gave me the following definition:  

> Matchers provide the granular tools for carrying out the assertions. They examine the data and verify that the data conforms to specific criteria.

Let's introduce matchers in the code. We can introduce a set of diverse matchers, each implementing the `Matcher` trait to work with specific data types.
The important part is that we will only be implementing matchers to deal with positive assertions.

```rust
pub trait Matcher<T> {
    fn test(&self, value: &T) -> MatcherResult;
}
```

`Matcher` is the base trait that works on any generic type `T`. It defines a method `test` that returns an instance of `MatcherResult`.

```rust
pub struct MatcherResult {
    passed: bool,
    failure_message: String,
    inverted_failure_message: String,
}
```

`MatcherResult` which will play a crucial role in inverting a matcher.

Let's implement `MembershipMatcher` for the `String` type. It should be capable of asserting:

- if the given string contains a digit.
- if the given string contains the specified character.

```rust
pub enum MembershipMatcher {
    ADigit,
    Char(char),
}

impl<T> Matcher<T> for MembershipMatcher
    where T: AsRef<str>
{
    fn test(&self, value: &T) -> MatcherResult {
        match self {
            MembershipMatcher::ADigit => MatcherResult::formatted(
                value.as_ref().chars().any(|ch| ch.is_numeric()),
                format!("{:?} should contain a digit", value.as_ref()),
                format!("{:?} should not contain a digit", value.as_ref()),
            ),
            MembershipMatcher::Char(ch) => MatcherResult::formatted(
                value.as_ref().chars().any(|source| &source == ch),
                format!("{:?} should contain the character {:?}", value.as_ref(), ch),
                format!("{:?} should not contain the character {:?}", value.as_ref(), ch),
            ),
        }
    }
}

pub fn contain_a_digit() -> MembershipMatcher {
    MembershipMatcher::ADigit
}

pub fn contain_a_character(ch: char) -> MembershipMatcher {
    MembershipMatcher::Char(ch)
}
```

A few things to observe:
- `MembershipMatcher` implements the trait `Matcher<T>` for any `T` which can be represented as string.
- We use enum to represent `MembershipMatcher` because it allows logical grouping of matchers and each enum variant can hold its own data.
- Each arm of the `match` expression returns an appropriate instance of `MatcherResult`.
- We also provide public functions to get appropriate instances of `MembershipMatcher`.
- `MembershipMatcher` only implements positive assertions.
- `MembershipMatcher` also knows about the error messages that should be returned if the assertion(s) using this matcher fails.

Time to add a few tests.

```rust
#[cfg(test)]
mod tests {
    use crate::{contain_a_character, contain_a_digit, Matcher};

    #[test]
    fn should_contain_a_digit() {
        let matcher = contain_a_digit();
        assert!(matcher.test(&"password@1").passed);
    }

    #[test]
    fn should_contain_a_char() {
        let matcher = contain_a_character('@');
        assert!(matcher.test(&"password@1").passed);
    }
}
```

It is a decent start to matchers but we still need to answer a few questions:
- How to connect assertions and matchers?
- How to invert a matcher?
- How to compose matchers?

### Connecting assertions and matchers - using blanket trait and trait object

We have **assertions** which serve as the cornerstone of the test cases, defining the exact expectations the code must fulfill.
They act as a contract, ensuring that each data type (/data structure) adheres to its intended behavior.

We also have **matchers** which provide the granular tools for carrying out the assertions. They examine the data and 
verify that the data conforms to specific criteria.

Let's take a look at the relationship between assertions and matchers.

<div class="align-center-exclude-width-change">
    <img src="/assertions-matchers.webp" alt="relationship between assertions and matchers"/>
</div>

We will have positive and negative assertions both of which use positive matchers. The bridge (*which is yet to be built*) will connect matchers with assertions
and invert a matcher if a negative assertion like `should_not_contain_a_digit` is invoked.

Let's build the bridge using [blanket trait](#leveraging-blanket-trait-implementation). 

```rust
pub trait Should<T> {
    fn should(&self, matcher: &dyn Matcher<T>);
}

impl<T> Should<T> for T {
    fn should(&self, matcher: &dyn Matcher<T>) {
        let matcher_result = matcher.test(self);
        if !matcher_result.passed {
            panic!("assertion failed: {}", matcher_result.failure_message);
        }
    }
}
```

We provide a trait `Should` which is implemented for any `T`. 

> We can also implement the trait `Should` for any `T: Assertion` where `Assertion` can be a base marker trait for all the assertions. 

The method `should` takes a matcher as a parameter, invokes the test method passing `self` as the argument and panics if the matcher fails. 

The method `should` takes a parameter `matcher: &dyn Matcher<T>` which is a trait object in Rust. The [trait object](https://doc.rust-lang.org/book/ch17-02-trait-objects.html) 
points to both an instance of a type implementing a trait and a table that is used to look up trait methods on that type at runtime.
We create a trait object by specifying either `&` reference or a `Box<T>` smart pointer, then the `dyn` keyword, and then specifying the relevant trait.

> When we use trait objects, Rust must use [dynamic dispatch](https://doc.rust-lang.org/book/ch17-02-trait-objects.html).

The expression `dyn Matcher<T>` refers to any `Matcher` of type `T` and is unsized. Rust compiler needs all the method parameters to have a fixed size,
hence we use `reference to any Matcher<T> -> matcher: &dyn Matcher<T>`.

Similarly, we can define a trait that inverts the given matcher.

```rust
pub trait ShouldNot<T> {
    fn should_not(&self, matcher: &dyn Matcher<T>);
}

impl<T> ShouldNot<T> for T {
    fn should_not(&self, matcher: &dyn Matcher<T>) {
        let matcher_result = matcher.test(self);
        let passed = !matcher_result.passed;
        if !passed {
            panic!(
                "assertion failed: {}",
                matcher_result.inverted_failure_message
            );
        }
    }
}
```

We can now refactor the methods `should_contain_a_digit` and `should_not_contain_a_digit` of the `MembershipAssertion` to use the `MembershipMatcher`.

```rust
impl<T> MembershipAssertion for T
    where T: AsRef<str> + Debug {
    fn should_contain_a_digit(&self) -> &Self {
        //the method should passes self (which in this case is &str) 
        //to the test method of the matcher.
        self.should(&contain_a_digit()); 
        self
    }
    
    fn should_not_contain_a_digit(&self) -> &Self {
        //the method should_not passes self (which in this case is &str) 
        //to the test method of the matcher.
        self.should_not(&contain_a_digit()); 
        self
    }
}
```

All the tests pass and we can now commit :). 

There is a question on matchers that needs to be answered. Should our matchers take the ownership of the extra data that they may need or should they hold a reference? 

Let's understand this.

Remember our traits `Should` and `ShouldNot` pass `&self` to the `test` method of the matcher. 
This means if we invoke the method `should_contain_a_digit` on the `String` type, the test method of our `MembershipMatcher` will receive
a reference to `String`.  

Let's consider that our string `MembershipMatcher` starts providing support for testing whether a string contains any of the given characters.
Should the matcher now hold a reference to a slice of char or a vector of char? 

**Option1**: `MembershipMatcher` provides an enum variant `AnyChars` which holds a reference to a slice of char.
```rust
pub enum MembershipMatcher<'a> {
    ADigit,
    Char(char),
    AnyChars(&'a [char])
}
```

**Option2**: `MembershipMatcher` provides an enum variant `AnyChars` which holds a vector of char.
```rust
pub enum MembershipMatcher {
    ADigit,
    Char(char),
    AnyChars(Vec<char>),
}
```

Time to discuss lifetimes.

### Matchers and lifetimes

Every reference in Rust has a lifetime, which is the stretch of the program for which the reference is valid. A **reference** can be treated as a **pointer managed by Rust**.
We have already seen a reference in the form of `&str`.

> Rust does not have nil or dangling references.

Consider the below code:

```rust
fn main() { 
    let reference: &i32;                | --- 'a starts            
    {                                   |
        let value: i32 = 10;            |    | --- 'b starts
        reference = &value;             |    | --- 'b ends
    }                                   |
    println!("{}", reference);          | --- 'a ends      
}
```

The code above creates a variable `reference` that refers to an `i32`. We use the lifetime annotations `'a` and `'b` to denote the lifetimes of
`reference` and `value`. Rust does not allow dangling references that means the variable `reference` can not outlive `value`. This can also be stated
as: "the lifetime of the variable `reference` must be less than or equal to the lifetime of the variable `value`".

The above code fails with the the following error:

```rust
    reference = &value;
                ^^^^^^ borrowed value does not live long enough
    
    Here, `value` is dropped as soon as the inner block ends, but `reference` lives 
    through the scope of the function. If this were permitted, Rust would end up with 
    dangling references.  
```

> Lifetimes in Rust ensure that all the references and the container objects holding references are always valid.   

Let's get back to our question: Should our matchers hold a reference to the extra data or take the ownership?

We will an attempt to design matcher using references (**Option1**), which means our matcher will now have lifetime annotation.

```rust
pub enum MembershipMatcher<'a> {
    ADigit,
    Char(char),
    AnyChars(&'a [char])
}
```

The `MembershipMatcher` holds a reference to a slice of char for some lifetime `'a`. This means:
- `MembershipMatcher` must live for the stretch of the program which is **at most** as big as the stretch defined by `'a`. 
- `MembershipMatcher` must not outlive the reference

This decision requires us to make the following changes:

```rust
impl<'a, T> Matcher<T> for MembershipMatcher<'a>
    where T: AsRef<str>
{
    fn test(&self, value: &T) -> MatcherResult {
        match self {
            //skipping ADigit and Char variants
            MembershipMatcher::AnyChars(chars) => MatcherResult::formatted(
                chars.iter().any(|ch| value.as_ref().contains(*ch)),
                format!("{:?} should contain any of the characters {:?}", value.as_ref(), chars),
                format!("{:?} should not contain any of the characters {:?}", value.as_ref(), chars),
            )
        }
    }
}

pub fn contain_a_digit<'a>() -> MembershipMatcher<'a> {
    MembershipMatcher::ADigit
}

pub fn contain_a_character<'a>(ch: char) -> MembershipMatcher<'a> {
    MembershipMatcher::Char(ch)
}

pub fn contain_any_characters<'a>(chars: &'a [char]) -> MembershipMatcher<'a> {
    MembershipMatcher::AnyChars(chars)
}
```

We have introduced lifetime annotation `'a` in the `impl`, the public methods `contain_a_digit`, `contain_a_character` and  `contain_any_characters`.
This lifetime annotation tells the compiler that `MembershipMatcher` lives for the duration defined by `'a`, which in turn is the lifetime
of the character slice. 

> The lifetime defined in the last method `contain_any_characters` can be removed. Historically, all the rust methods with reference(s) as 
> parameter(s) required the developers to specify lifetime annotation(s). With time, rust developers saw some some common patterns and created
> a set of lifetime elision rules. One of those rules states: if a function accepts a single reference parameter, then the
> lifetime of that reference will be assigned as the lifetime of the return value.

We can now add a test.

```rust
#[cfg(test)]
mod matcher_tests {

    #[test]
    fn should_contain_any_chars() {
        let matcher = contain_any_characters(&['@', '#', '.']);
        assert!(matcher.test(&"password@1").passed);
    }
}
```

This works fine. 

Let's play with matchers and lifetimes for a bit by adding a few tests.

```rust
#[cfg(test)]
mod tests {
    #[test]
    fn should_contain_any_chars() {
        let matcher;                                            
        {                                                       
            let chars = &['@', '#', '.'];                       
            matcher = contain_any_characters(chars);            
        }                                                           
        assert!(matcher.test(&"password@1").passed);            
    }
}
```

We declare a `matcher` variable in the outer block, which is initialized in the inner block. We would like the character slice to live at least as long as the matcher, 
otherwise `matcher` would point to a dropped character slice. 

It might appear that `chars` will be dropped as soon as the inner block ends, and the above code will not compile. 
However, Rust extends the lifetime of `chars` to match the lifetime of the variable `matcher` which is till the end of the function. 
The above code compiles just fine.

Let's take another example where Rust results in compilation error. It is a slight change in the test, but an important one.

```rust
#[cfg(test)]
mod tests {
    #[test]
    fn should_contain_any_chars_compiler_error() {
        let matcher;
        {
            let chars = ['@', '#', '.'];
            matcher = contain_any_characters(&chars);
                                             ^^^^^^ borrowed value does not live long enough
        }
        assert!(matcher.test(&"password@1").passed); - chars dropped here while still borrowed
    }
}
```

Here, we declare `chars` as an array, and its reference is passed to the function `contain_any_characters`. Rust will drop `chars` array at the end 
of the inner block, however `matcher` outlives the reference. Rust does not allow dangling references and thus it results in a compilation error. 

Question, how do we decide if matchers should have a reference of the extra data or own the data?

We need to answer a few questions to take this decision:
1. This crate will be used in testing (unit/integration/functional/any other). Is it ok if a matcher takes the ownership of extra data?
2. Do you care about the consistency of ownership vs borrow concept? Do you want all the matchers to refer to extra data or take ownership? 
3. Are you going to build a concept that [combines various matchers](#matcher-composition) and can the lifetimes of various matchers cause any problems?

I recently finished an assertions crate called [clearcheck](https://github.com/SarthakMakhija/clearcheck) and I decided to have matchers own their data. My decision was primarily
based on point 1.  

### Matcher composition : using trait objects

We now have matcher as a citizen in the code. I think it is worth building a concept that allows us to combine various matchers using operators like: 
**and**, **or**. 

We can introduce an abstraction `Matchers` that contains a collection of objects which implement `Matcher<T>` trait.

```rust
enum Kind {
    And,
    Or,
}

pub struct Matchers<T, M: Matcher<T>> {
    matchers: Vec<M>,
    kind: Kind,
    inner: PhantomData<T>,
}
```

Our first attempt is to create a `Matchers` abstraction that holds a vector of `M`, where `M` is a `Matcher` that operates on a `T`.

To understand the issue with this design, we need to understand how generics are processed in Rust. In Rust, generics undergo [monomorphization](https://rustc-dev-guide.rust-lang.org/backend/monomorph.html) which means the compiler 
produces a different copy of the generic code for each concrete type needed. This means if the generic type `Option<T>` is used with `f64` and `i32`, 
the compiler will produce two copies of `Option`, which would be similar to `Option_f64` and `Option_i32`.

Given our definition of `Matchers`, Rust will emit different copies of `Matchers` for each concrete type of `Matcher`. This also means that Rust
will not allow us to combine different types of matcher objects using generics, even if all of them implement the `Matcher<T>` trait for the same `T`.

So, we need a different way to combine matchers. We have already seen [trait objects](#connecting-assertions-and-matchers---using-blanket-trait-and-trait-object),
which point to both an instance of a type implementing a trait and a table that is used to look up trait methods on that type at runtime.
We create a trait object by specifying either `&` reference or a `Box<T>` smart pointer, then the `dyn` keyword, and then specifying the relevant trait.
Trait objects enable storing objects (inside a struct) regardless of their specific types, as long as they fulfill the required behavior specified by a trait.

We can now represent `Matchers` using a vector of trait objects, which implement the `Matcher<T>` trait.

```rust
pub struct Matchers<T> {
    matchers: Vec<Box<dyn Matcher<T>>>,
    kind: Kind,
}
```

Let's now implement a builder to create a `Matchers` object.

(*I am only going to describe the **and** operator, and **positive assertions***. 
You can refer the code [here](https://github.com/SarthakMakhija/clearcheck/blob/main/src/matchers/compose/mod.rs) for composition with inverted assertions.)

```rust
pub struct MatchersBuilder<T> {
    matchers: Vec<Box<dyn Matcher<T>>>,
}

impl<T: Debug> MatchersBuilder<T> {
    pub fn start_building(matcher: Box<dyn Matcher<T>>) -> Self {
        MatchersBuilder {
            matchers: vec![matcher]
        }
    }

    pub fn push(mut self, matcher: Box<dyn Matcher<T>>) -> Self {
        self.matchers.push(matcher);
        self
    }

    pub fn combine_as_and(self) -> Matchers<T> {
        Matchers::and(self.matchers)
    }
}
```

`MatchersBuilder` allows composing multiple matchers and it can be used as following:

```rust
MatchersBuilder::start_building(Box::new(contain_a_digit()))
    .push(Box::new(contain_any_characters(&['#', '&'])))
    .push(Box::new(contain_a_character('0')))
    .combine_as_and()
```

Finally, `Matchers` is going to implement the `Matcher<T>` trait. This is what makes the crate really powerful. 
Anyone can create their custom matchers and use them in the assertion(s) of their choice.

```rust
impl<T: Debug> Matcher<T> for Matchers<T> {
    fn test(&self, value: &T) -> MatcherResult {
        let results = self
            .matchers
            .iter()
            .map(|matcher| matcher.test(value))
            .collect::<Vec<_>>();

        match self.kind {
            Kind::And => MatcherResult::formatted(
                results.iter().all(|result| result.passed),
                messages(
                    &results,
                    |result| !result.passed,
                    |result| result.failure_message.clone(),
                ),
                messages(
                    &results,
                    |result| result.passed,
                    |result| result.inverted_failure_message.clone(),
                ),
            ),
            //Kind::Or skipped
        }
    }
}

fn messages<P, M>(results: &[MatcherResult], predicate: P, mapper: M) -> String
    where
        P: Fn(&&MatcherResult) -> bool,
        M: Fn(&MatcherResult) -> String,
{
    results
        .iter()
        .filter(predicate)
        .map(mapper)
        .collect::<Vec<_>>()
        .join("\n")
}
```

The `test` method runs all the matchers and collects `MatcherResult`s. It then produces an instance of `MatcherResult` based on the `Kind`.

Let's see the concept of custom matchers in action.

```rust
#[cfg(test)]
mod custom_string_matchers_tests {
    use std::fmt::Debug;

    use crate::{contain_a_character, contain_a_digit, contain_any_characters, Matchers, MatchersBuilder, Should};

    fn be_a_valid_password<T: AsRef<str> + Debug>() -> Matchers<T> {
        MatchersBuilder::start_building(Box::new(contain_a_digit()))
            .push(Box::new(contain_any_characters(&['#', '&', '@'])))
            .push(Box::new(contain_a_character('0')))
            .combine_as_and()
    }

    trait PasswordAssertion {
        fn should_be_a_valid_password(&self) -> &Self;
    }

    impl PasswordAssertion for &str {
        fn should_be_a_valid_password(&self) -> &Self {
            self.should(&be_a_valid_password());
            self
        }
    }
    
    #[test]
    fn should_be_a_valid_password() {
        let password = "P@@sw0rd9082";
        password.should_be_a_valid_password();
    }
}
```

This involves the following:
- Creating a custom password matcher.
- Creating a custom password assertion.
- Implementing the password assertion for `&str`.
- Leveraging the custom matcher in the assertion using the built-in trait `Should`.

We have arrived at the end of the article. I hope it was worth your time. 

I have an assertions crate called [clearcheck](https://github.com/SarthakMakhija/clearcheck). Do take a look if it excites you.

### References

- [The Rust Programming Language](https://doc.rust-lang.org/book/)
- [Programming Rust](https://www.oreilly.com/library/view/programming-rust-2nd/9781492052586/)
- [String types](https://stackoverflow.com/questions/24158114/what-are-the-differences-between-rusts-string-and-str)
