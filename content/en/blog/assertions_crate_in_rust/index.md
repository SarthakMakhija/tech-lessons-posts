---
author: "Sarthak Makhija"
title: "Building an assertions crate in Rust"
date: 2024-01-21
description: ""
tags: ["Rust", "Assertions", "Elegant-assertions"]
thumbnail: /aws-lambda-virtual-podcast.webp
caption: ""
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

### Let's get started - beginning with extension methods

The first thing that we need is provide custom methods on the built-in types. In the password validation example, we have methods like
`should_contain_all_characters` and `should_contain_a_digit` over `string` (or `&str`).

Such methods are called "extension" methods and [Wikipedia](https://en.wikipedia.org/wiki/Extension_method#:~:text=Extension%20methods%20are%20features%20of,an%20equally%20safe%20manner%2C%20however.) defines extension method as
"a method added to an object after the original object was compiled". **In Rust, extension methods are added using traits.**

Let's start by defining a trait.

```rust
pub trait MembershipAssertion {
    fn should_contain_a_digit(&self) -> &Self;
    fn should_not_be_empty(&self) -> &Self;
}
```

`MembershipAssertion` defines methods: `should_contain_a_digit` and `should_not_be_empty`, both of which returns reference to `Self` to allow chaining.

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

### Leverage blanket trait implementation

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

### Matchers and lifetimes

### Matcher composition

### Conclusion 

