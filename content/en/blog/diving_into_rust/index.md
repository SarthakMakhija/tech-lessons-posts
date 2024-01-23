---
author: "Sarthak Makhija"
title: "Diving into Rust by building an assertions crate"
date: 2024-01-21
description: "
As Rust projects grow in size and complexity, the need for sophisticated error handling tools becomes ever more pressing. 
Traditional methods like panics and asserts, while useful, can be limited and cumbersome.
Let's build an assertions crate that offers elegant and powerful assertions, while simultaneously diving into the diverse landscape of Rust features. 
"
tags: ["Rust", "Assertions", "Elegant-assertions", "Clearcheck"]
thumbnail: /diving-into-rust.jpg
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

### Getting started - beginning with extension methods

The first thing that we need is provide custom methods on the built-in types. In the password validation example, we have methods like
`should_contain_all_characters` and `should_contain_a_digit` over `string` (or `&str`).

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

Let's implement the trait for a reference to string slice (`&str`). *(I will only implement should_contain_a_digit for the article).*

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

Time to add a few tests. *(I will only show a couple of tests for the article).*

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

We have implemented the `MembershipAssertion` trait for `&str` type. We also need the methods of the `MembershipAssertion` trait for the `String` type.

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

We have simply duplicated the method `should_contain_a_digit` for `&str` and `String` and verified that it works for both the types. 

Let's make an attempt to remove the duplication.

We can convert the `String` type to a `reference to string slice` and invoke the respective methods on `&str`. With this approach, the implementation
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

Let's revisit our `MembershipAssertion`.

```rust
pub trait MembershipAssertion {
    fn should_contain_a_digit(&self) -> &Self;
    fn should_not_be_empty(&self) -> &Self;
}
```

The idea is to provide an implementation of `MembershipAssertion` for all the types which can be represented as string. Rust provides various such types:

- String: //TODO 
- str:    //TODO
- &str:   //TODO

It will be great to implement `MembershipAssertion` for any `T`, where `T` that it can be represented as `String` (or `&str`).

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

Jim Blandy in the book [Programming Rust](https://www.oreilly.com/library/view/programming-rust-2nd/9781492052586/?_gl=1*14ve435*_ga*MTQxOTMyMjU5Ni4xNjg5MjQ4Mjgy*_ga_092EL089CH*MTcwNTgzNjI5OC4zOS4xLjE3MDU4MzYzMDQuNTQuMC4w) says: when a type implements `AsRef<U>`, that means you can borrow a `reference of U` from the type.
This means, if a type implements `AsRef<str>` we should be able to borrow a `reference of str` from that type.

In Rust, both `String` and `&str` type implements `AsRef<str>` which means all the methods of `MembershipAssertion` are now available on `String`
and `&str`. Please find the reference [here](https://doc.rust-lang.org/std/convert/trait.AsRef.html#:~:text=Since%20both%20String%20and%20%26str,accept%20both%20as%20input%20argument). 

We need the `T` to implement `Debug` trait because we are formatting it in the `panic` macro.

Quick revisit: 
- Rust allows implementing a trait for any `T`. This is called *blanket trait implementation*. In our case, we are implementing `MembershipAssertion`
for a constrained `T`.
- The trait `AsRef` provides a lot of flexibility in the function signature. The method [open](https://doc.rust-lang.org/std/fs/struct.File.html#method.open) in
Rust [File](https://doc.rust-lang.org/std/fs/struct.File.html) type takes `AsRef<Path>`, which means any `P` which can be represented as a [Path](https://doc.rust-lang.org/std/path/struct.Path.html)
can be passed as an argument.
```rust
// open method from Rust' file type.
pub fn open<P: AsRef<Path>>(path: P) -> Result<File> {..}
```

With the introduction of `AsRef<str>` change, we have taken care of the duplication between implementations of `MembershipAssertion`, and we can
now remove our original implementation of `MembershipAssertion` for `String` and `&str` types.

### Leveraging blanket trait implementation

This crate will also offer assertions related to ordered comparison (greater than, less than, greater than equal to, less than equal to) for various types. Let's
provide such a trait.

```rust
pub trait OrderedAssertion<T: PartialOrd> {
    fn should_be_greater_than(&self, other: T) -> &Self;
    fn should_not_be_greater_than(&self, other: T) -> &Self;
    // other methods
}
```
The trait `OrderedAssertion` is generic over `T: PartialOrd`. [PartialOrd](https://doc.rust-lang.org/std/cmp/trait.PartialOrd.html) trait allows ordered comparison (>, <. >=, <=) between types.

We have an option of implementing `OrderedAssertion` for various types like `i8`, `i16`, `i32`, `&str` etc. or we can implement `OrderedAssertion`
for a constrained `T`.

Let's attempt implementing `OrderedAssertion` for any type `T` that implements `PartialOrd`.

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

The advantage of implementing a trait for any `T` (with or without constraint() is obvious. It removes duplication!!! 

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

There is still subtle duplication between the implementations of `should_be_greater_than` and `should_not_be_greater_than`. 

Let's take a closer look to understand the duplication:
- In comparison: `self <= &other` and `self > &other`, both the methods are doing comparisons which are inverted.
- In the panic message: `panic!("{:?} should be greater than {:?}", self, other)` and `panic!("{:?} should not be greater than {:?}", self, other)`,
both the methods are causing panic with messages which are inverted.

The same form of duplication will be observed between different method pairs: 
- `should_be_empty`, `should_not_be_empty` 
- `should_be_greater_than`, `should_not_be_greater_than`
- etc

This duplication is caused because of the change in the condition for examining the data: *self must be **less than** other*, *self must be **greater than** other*.

We can deal with the duplication by introducing an abstraction that:
- Can **examine the data** and **verify** that the data conforms to **specific criteria**.
- Can invert its behavior so that we **won't** be required to write any **conditional code** for implementing **negative assertions**. 

The question is what would be the name of such an abstraction? In the world of assertions, such an object is called matcher. I asked [bard](https://bard.google.com/chat) to define *Matcher*? It gave me the following definition:  

> Matchers provide the granular tools for carrying out the assertions. They examine the data and verify that the data conforms to specific criteria.

Let's introduce matchers in the code. We can introduce a set of diverse matchers, each implementing the `Matcher` trait to work with specific data types, ensuring flexibility and precision in our checks.
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

Let's implement `MembershipMatcher` for string. It should be capable of testing:

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
- `MembershipMatcher` implements `Matcher<T>` for any `T` which can be represented as string.
- We use enum to represent `MembershipMatcher` because it allows logical grouping of matchers and each enum variant can hold its own data.
- Each arm of the `match` expression returns an appropriate instance of `MatcherResult`.
- We also provide public functions to return appropriate instances of `MembershipMatcher`.
- `MembershipMatcher` only implements positive assertions, thus taking the duplication out. *However, we still need to invert matchers.*
- `MembershipMatcher` also knows about the error messages that should be returned if the assertion(s) using this matcher fails.

Let's add a few tests.

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

### Connecting assertions and matchers - using blanket trait and dyn trait 

We have **assertions** which serve as the cornerstone of the test cases, defining the exact expectations the code must fulfill.
They act as a contract, ensuring that each data type (/data structure) adheres to its intended behavior.

We also have **matchers** which provide the granular tools for carrying out the assertions. They examine the data and 
verify that the data conforms to specific criteria.

Let's take a look at the relationship between assertions and matchers.

<div class="align-center-exclude-width-change">
    <img src="/assertions-matchers.png" alt="relationship between assertions and matchers"/>
</div>

We will have positive and negative assertions, with only positive matchers. The bridge (*which is yet to be built*) will connect matchers with assertions
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

The method `should` takes a parameter `matcher: &dyn Matcher<T>`which is read as "reference to any object of type Matcher which takes T as a generic parameter".
The expression `dyn Matcher<T>` refers to any `Matcher` of type `T` and is unsized. Rust compiler needs all the method parameters to have a fixed size,
hence we use `reference to any Matcher<T> -> matcher: &dyn Matcher<T>`. A reference (or a borrow) is a simple pointer in Rust which can never be null or dangling.

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
        self.should(&contain_a_digit());
        self
    }
    
    fn should_not_contain_a_digit(&self) -> &Self {
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

//TODO: Introduce lifetimes

Let's take option 1, where the `MembershipMatcher` provides an enum variant `AnyChars` which holds a reference to a slice of char.

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
                format!("{:?} should not any of contain the characters {:?}", value.as_ref(), chars),
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
> parameter(s) required the developers to specify lifetime annotation(s). With time, rust developers realized some common patterns and created
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

This works fine. Let's play with Rust lifetimes for a bit by adding a few tests.

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

We declare a `matcher` variable in the outer block which is initialized in the inner block. We would like the character slice to live at least as long as the matcher, 
otherwise matcher would pointer to an already dropped character slice. 

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

Here, we declare `chars` as an array, and its reference is passed to the function `contain_any_characters`. Rust will drop `chars` at the end 
of the inner block, however matcher outlives the reference. Rust does not allow dangling references and thus it results in a compilation error. 

### Matcher composition

### Conclusion (with clearcheck reference) 

