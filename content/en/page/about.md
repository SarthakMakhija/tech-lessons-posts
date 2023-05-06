---
author: Sarthak Makhija
title: About Me
date: 2023-02-24
description:
keywords: ["about-us", "contact"]
type: about
---

<style>
 .social{
    display: inline-block;
    text-align: left;
    width: 100%;
    color: #a6a6a6;
    font-size: .9em;
}
</style>

<div class="self-container">
    <p><img class="self-image" alt="Sarthak Makhija" src="/self.png"></p> 
    <p class="self">Sarthak Makhija</p>
</div>
I am an application developer at ThoughtWorks. I had worked with Citigroup and TCS before I joined ThoughtWorks.

## My Projects

I love working on my projects in my free time. Some of my projects include –

🔹 **[bitcask](https://github.com/SarthakMakhija/bitcask)**

This is the golang implementation of Riak's [bitcask](https://riak.com/assets/bitcask-intro.pdf) paper. This project is for educational purposes only. The idea is to provide a reference implementation of bitcask to help anyone interested in storage engines understand the basics of building a persistent key-value storage engine.

🔹 **[goselect](https://github.com/SarthakMakhija/goselect)**

goselect provides SQL-like ‘select’ interface for files. This means one can execute a select query like:

`select name, path, size from . where or(like(name, result.*), eq(isdir, true)) order by 3 desc`
to get the filename, file path and size of all the files that are directories or their names begin with the term result. This query orders the results by size in descending order.

I created goselect to understand the following:
- **Parsing**: The parsing pipeline typically involves a lexer, a parser and an AST.
- **Recursive descent parser**
- **Representation of functions in the code**. Functions like `lower`, `upper` and `trim` take a single parameter, functions like `now` and `currentDate` take zero parameters, whereas functions like `concat` take a variable number of parameters.
- **Execution of scalar functions like `lower`, `upper` and `substr`**. These functions are stateless and run for each row. They may take a parameter and return a value for each row. These functions can also involve nesting. For example, `lower(substr(ext, 1))`.
- **Execution of aggregation functions like `average`, `min`, `max` and `countDistinct`**. These functions run over a collection of rows and return an ultimate value. These functions can also involve nesting. For example, `average(count())`.
- **Execution of nested scalar and aggregation functions like `countDistinct(lower(name))`**. Here, the function `lower(name)` runs for each row, whereas `countDistinct` runs over a collection of rows.

🔹 **[Gamifying Refactoring](http://gamifying-refactoring.github.io/)**

Refactoring is fun to learn and practice. It is even more fun to understand it together by playing a game.

All you got to do is - identify code smells, **justify** each of them by going beyond **ilities**, finish all of this in a fixed time and get points for your team. Learn and have fun. This is the idea behind “Gamifying Refactoring”.

🔹 **[Data-anon](https://github.com/dataanon/data-anon)**

Data Anonymization tool helps build **anonymized production data dumps**, which can be used for performance testing, security testing, debugging and development. We implemented this tool in Kotlin, which works with Java 8 & Kotlin.

🔹 **[Flips](https://github.com/Feature-Flip/flips)**

Flips is an implementation of [Feature Toggles](https://martinfowler.com/articles/feature-toggles.html) pattern for Java. Feature Toggles are a powerful technique, allowing teams to change system behaviour without changing the code.

The idea behind Flips is to let the users **implement toggles with minimum configuration and coding**. This library should work with Java8, Spring Core / Spring MVC / Spring Boot.

I have a separate [blog](https://tech-lessons.in/blog/flips_feature_flipping_for_java/) about this library.

## Current projects

Currently, I am implementing an [LFU based cache](https://github.com/SarthakMakhija/cached) in rust. 

## Next project
My current interest is in *Designing storage engines and databases*, and I plan to build a key/value storage engine in Rust. The idea is to create a write-optimized storage engine using an LSM tree and provide read-optimized paths.
Some ideas that I would like to explore are:
1. Thread-per-core programming model 
2. Separating values from keys: [WiscKey](https://www.usenix.org/system/files/conference/fast16/fast16-papers-lu.pdf)
3. Explore [io_uring](https://unixism.net/loti/what_is_io_uring.html) and [glommio](https://github.com/DataDog/glommio)
4. Cache bloom filters in memory
5. Cache level-0 SSTables in memory

## Let's connect
<div class="flex gap-x-3 flex-wrap gap-y-2">
    <a
      href="https://twitter.com/MakhijaSarthak"
      target="_blank"
      rel="noopener"
      aria-label="Twitter"
      class="p-1 inline-block rounded-full border border-transparent text-gray-500 hover:text-gray-800 hover:border-gray-800 cursor-pointer transition-colors dark:text-gray-600 dark:hover:border-gray-300 dark:hover:text-gray-300"
    >
      <svg
        xmlns="http://www.w3.org/2000/svg"
        width="34"
        height="34"
        viewBox="0 0 24 24"
        stroke-width="1.5"
        stroke="currentColor"
        fill="none"
        stroke-linecap="round"
        stroke-linejoin="round"
      >
        <path stroke="none" d="M0 0h24v24H0z" fill="none" />
        <path
          d="M22 4.01c-1 .49 -1.98 .689 -3 .99c-1.121 -1.265 -2.783 -1.335 -4.38 -.737s-2.643 2.06 -2.62 3.737v1c-3.245 .083 -6.135 -1.395 -8 -4c0 0 -4.182 7.433 4 11c-1.872 1.247 -3.739 2.088 -6 2c3.308 1.803 6.913 2.423 10.034 1.517c3.58 -1.04 6.522 -3.723 7.651 -7.742a13.84 13.84 0 0 0 .497 -3.753c-.002 -.249 1.51 -2.772 1.818 -4.013z"
        />
      </svg>
    </a>
    <a
      href="https://www.linkedin.com/in/sarthak-makhija-7a165a55"
      target="_blank"
      rel="noopener"
      aria-label="LinkedIn"
      class="p-1 inline-block rounded-full border border-transparent text-gray-500 hover:text-gray-800 hover:border-gray-800 cursor-pointer transition-colors dark:text-gray-600 dark:hover:border-gray-300 dark:hover:text-gray-300"
    >
      <svg
        xmlns="http://www.w3.org/2000/svg"
        width="34"
        height="34"
        viewBox="0 0 24 24"
        stroke-width="1.5"
        stroke="currentColor"
        fill="none"
        stroke-linecap="round"
        stroke-linejoin="round"
      >
        <path stroke="none" d="M0 0h24v24H0z" fill="none" />
        <rect x="4" y="4" width="16" height="16" rx="2" />
        <line x1="8" y1="11" x2="8" y2="16" />
        <line x1="8" y1="8" x2="8" y2="8.01" />
        <line x1="12" y1="16" x2="12" y2="11" />
        <path d="M16 16v-3a2 2 0 0 0 -4 0" />
      </svg>
    </a>
    <a
      href="https://github.com/SarthakMakhija"
      target="_blank"
      rel="noopener"
      aria-label="GitHub"
      class="p-1 inline-block rounded-full border border-transparent text-gray-500 hover:text-gray-800 hover:border-gray-800 cursor-pointer transition-colors dark:text-gray-600 dark:hover:border-gray-300 dark:hover:text-gray-300"
    >
      <svg
        xmlns="http://www.w3.org/2000/svg"
        width="34"
        height="34"
        viewBox="0 0 24 24"
        stroke-width="1.5"
        stroke="currentColor"
        fill="none"
        stroke-linecap="round"
        stroke-linejoin="round"
      >
        <path stroke="none" d="M0 0h24v24H0z" fill="none" />
        <path
          d="M9 19c-4.3 1.4 -4.3 -2.5 -6 -3m12 5v-3.5c0 -1 .1 -1.4 -.5 -2c2.8 -.3 5.5 -1.4 5.5 -6a4.6 4.6 0 0 0 -1.3 -3.2a4.2 4.2 0 0 0 -.1 -3.2s-1.1 -.3 -3.5 1.3a12.3 12.3 0 0 0 -6.2 0c-2.4 -1.6 -3.5 -1.3 -3.5 -1.3a4.2 4.2 0 0 0 -.1 3.2a4.6 4.6 0 0 0 -1.3 3.2c0 4.6 2.7 5.7 5.5 6c-.6 .6 -.6 1.2 -.5 2v3.5"
        />
      </svg>
    </a>
</div>