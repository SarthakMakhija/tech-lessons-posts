---
author: Sarthak Makhija
title: My projects
date: 2024-01-20
description:
keywords: ["projects"]
type: page
---

## My Projects

I love working on my projects in my free time.

### 🔹 [Clearcheck](https://github.com/SarthakMakhija/clearcheck)

Write expressive and elegant assertions with ease!

**clearcheck** is designed to make assertion statements in **Rust** as clear and concise as possible.

It allows chaining multiple assertions together for a fluent and intuitive syntax, leading to more self-documenting test cases.

```rust
let pass_phrase = "P@@sw0rd1 zebra alpha";
pass_phrase.should_not_be_empty()
    .should_have_at_least_length(10)
    .should_contain_all_characters(vec!['@', ' '])
    .should_contain_a_digit()
    .should_not_contain_ignoring_case("pass")
    .should_not_contain_ignoring_case("word");
```

### 🔹 [Blast](https://github.com/SarthakMakhija/blast)

**blast** is a load generator for TCP servers, especially if such servers maintain persistent connections. It is implemented in **golang**.
It is used in my current project to do the load testing of the distributed key/value storage engine that we are building.

**blast** provides the following features:

- Support for sending N requests to the target server.
- Support for reading N total responses from the target server.
- Support for reading N successful responses from the target server.
- Support for customizing the load duration. By default, blast runs for 20 seconds.
- Support for sending N requests to the target server with the specified concurrency level.
- Support for establishing N connections to the target server.
- Support for specifying the connection timeout.
- Support for specifying requests per second (also called throttle).
- Support for printing the report.

```shell
./blast -n 200000 -c 100 -conn 10 -f ./payload localhost:8989
```

The above command sends 200000 requests with the payload defined in the file `payload`, over 10 TCP connections using 100 concurrent workers.

### 🔹 [CacheD](https://github.com/SarthakMakhija/cached)

**CacheD** is a [high performance](https://github.com/SarthakMakhija/cached/tree/main/benches/results), [LFU](https://dgraph.io/blog/refs/TinyLFU%20-%20A%20Highly%20Efficient%20Cache%20Admission%20Policy.pdf) based in-memory cache in **Rust** inspired by [Ristretto](https://github.com/dgraph-io/ristretto).

```rust
#[tokio::test]
async fn put_a_key_value() {
    let cached = CacheD::new(ConfigBuilder::new(COUNTERS, CAPACITY, CACHE_WEIGHT).build());
    let acknowledgement =
            cached.put("topic", "LFU cache").unwrap();
     
     let status = acknowledgement.handle().await;
     assert_eq!(CommandStatus::Accepted, status);
    
     let value = cached.get(&"topic");
     assert_eq!(Some("LFU cache"), value);
}
```

### 🔹 [Goselect](https://github.com/SarthakMakhija/goselect)

goselect provides SQL-like ‘select’ interface for files. This means one can execute a select query like:

`select name, path, size from . where or(like(name, result.*), eq(isdir, true)) order by 3 desc`
to get the filename, file path and size of all the files that are either directories or their names begin with the term result.

This tool is implemented in **golang**.

### 🔹 [Gamifying refactoring](http://gamifying-refactoring.github.io/)

Refactoring is fun to learn and practice. It is even more fun to understand it together by playing a game.

I created the idea of Gamifying refactoring which is run as a game (/mini workshop) in TW.

The idea behind this game is to **identify** code smells, **justify** each of them by going beyond **ilities**, finish all of this in a fixed time and win points for your team.

### 🔹 [Data-anon](https://github.com/dataanon/data-anon)

Data Anonymization tool helps build **anonymized production data dumps**, which can be used for performance testing, security testing, debugging and development. This tool is implemented in **Kotlin** and works with Java 8 & Kotlin.

```kotlin

fun main(args: Array<String>) {
    // define your database connection settings 
    val source = DbConfig("jdbc:h2:tcp://localhost/~/movies_source", "sa", "")
    val dest = DbConfig("jdbc:h2:tcp://localhost/~/movies_dest", "sa", "")

    // choose Whitelist or Blacklist strategy for anonymization
    Whitelist(source, dest)
            .table("MOVIES") {              // start with table                                
                where("GENRE = 'Drama'")    // allows selection only desired rows (optional)
                limit(1_00_000)             // useful for testing                 (optional)

                // pass through fields, leave it as is (do not anonymize)
                whitelist("MOVIE_ID")

                // field by field decide the anonymization strategy
                anonymize("GENRE").using(PickFromDatabase<String>(source,"SELECT DISTINCT GENRE FROM MOVIES"))
                anonymize("RELEASE_DATE").using(DateRandomDelta(10))

                // write your own in-line strategy
                anonymize("TITLE").using(object: AnonymizationStrategy<String>{
                    override fun anonymize(field: Field<String>, record: Record): String = "MY MOVIE ${record.rowNum}"
                })
            }
            // continue with multiple tables
            .table("RATINGS") {
                whitelist("MOVIE_ID","USER_ID","CREATED_AT")
                anonymize("RATING").using(FixedDouble(4.3))
            }
            .execute()
}
```

### 🔹 [Flips](https://github.com/Feature-Flip/flips)

Flips is an implementation of [Feature Toggles](https://martinfowler.com/articles/feature-toggles.html) pattern for Java. 

Feature Toggles are a powerful technique that allow teams to change system behaviour without changing the code.

The idea behind Flips is to let the users **implement toggles with minimum configuration and coding**. This library is implemented in **Java** and uses **Spring MVC**.

```java
@Component
class EmailSender{

  @FlipOnEnvironmentProperty(property="feature.send.email", expectedValue="true")
  public void sendEmail(EmailMessage emailMessage){
  }
}
```

*Feature sendEmail is enabled if the property `feature.send.email` is set to `true`.*

I have a separate [blog](https://tech-lessons.in/blog/flips_feature_flipping_for_java/) about this library.
