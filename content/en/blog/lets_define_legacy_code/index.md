---
author: "Sarthak Makhija"
title: "Let's define Legacy code"
date: 2018-04-10
description: "I have been having sleepless nights trying to add features in the code that we acquired from other company. I am dealing with the purest form of Legacy code. I am having a real hard time dealing with tangled, unstructured code that I have to work with, but I don’t understand a bit. Legacy code!. Let's understand what is Legacy code."
tags: ["Legacy code", "Boy Scout Rule"]
thumbnail: /legacy-code.webp
caption: "Photo by Markus Spiske on Pexels"
---
"I have been having sleepless nights trying to add features in the code that we acquired from other company. I am dealing with the purest form of *Legacy code*."

"I am having a real hard time dealing with tangled, unstructured code that I have to work with, but I don’t understand a bit. *Legacy code*"

*Legacy code* is a term which probably has a lot of different definitions like: code acquired from someone else, code written by someone else, code that is hard to understand or code written in outdated technologies. Whatever be the definition, most of us believe *Legacy code is Scary.*

**Q**uestion> How would you define legacy code?

### Defining Legacy code
Michael Feathers in his book “Working Effectively with Legacy code” defines legacy code as the code without tests.

code without tests is a bad code. It doesn’t matter how well it is written; how well it is structured; how well it is encapsulated. Without tests there is no way to tell if our code is getting better or worse.

Well, a slightly modified version of this definition is “code without unit tests is called legacy code”. It is always better to have tests as close to the code as possible *unit tests > integration tests > UI tests*. So, it would not be unfair to call a code without unit tests a *legacy code*.

### Working with Legacy code

**Q**uestion> What approach will you take if you were to make a change in legacy code?

Most of us might say, “I will make the change and call it a day, why bother about improving the code”. Rationale behind this thought process could be:
- I don’t have enough time to refactor the code, I would prefer making a change and completing my story
- Why risk changing the structure of the code that has been running on the production environment for a long time
- What is the overall benefit of refactoring legacy code

Michael Feathers calls this style of making a change as *Edit and Pray*. You plan and make your changes and when you are done, you pray and pray harder to get your changes right.
With this style, one can only contribute to increasing the Legacy code.

<div class="align-center">
<img title="Legacy code" src="/legacy-code.jpeg" alt="Legacy code" />
</div>

There is a different style of making changes in the legacy code and that is *Cover and Modify*. Build a safety net, make changes in the system, let the safety net provide feedback and work on those feedbacks.

It can be safely assumed that *Cover and Modify* is a way to go to deal with Legacy code.

**Q**uestion> Should you even spend time writing tests in a legacy code or even think about refactoring a legacy code?

### The Boy Scout Rule
> The idea behind the Boy Scout Rule, as stated by Uncle Bob, is fairly simple: Leave the code cleaner than you found it! Whenever you touch an old code, you should clean it properly. Do not just apply a shortcut solution that will make the code more difficult to understand but instead treat it with care. It’s not enough to write code well, the code has to be kept clean over time.

We get a very strong message when Boy Scout rule is applied to legacy code: *leave a trace of understanding behind you for others to follow*, which means we will refactor the code to make it more understandable. And in order to refactor, we will build  safety net around it.
Now that we understand that we can not take shortcuts, the only option that is left with us is to write some tests, refactor code and then proceed with the development.

**Q**uestions>
- Which tests should we write?
- How much should we refactor?

### Which Tests To Write
In nearly every legacy system, what the system does is more important than what it is supposed to do.
> Characterization Tests, the tests that we need when we want to preserve the behavior are called as characterization tests. A characterization test is a test that characterizes the actual behavior of a piece of code. There’s no “Well, it should do this” or “I think it does that”. These tests document the current behavior of the system.

### Writing Characterization Test
A *Characterization Test* by definition documents the current behavior of the system the exact same way it is running on the production environment.

Let’s write a characterization test for a `Customer` class that generates text statement for some movies rented by a customer.
```java
import static com.code.legacy.movie.MovieType.CHILDREN;
import static org.junit.Assert.assertEquals;

public void shouldGenerateTextStatement(){
    Customer john          = new Customer("John");
    Movie    childrenMovie = new Movie("Toy Story", CHILDREN);   
    int      daysRented    = 3;
    Rental   rental        = new Rental(childrenMovie, daysRented);
    john.addRental(rental);

    String statement = john.generateTextStatement();
    assertEquals("", statement);
}
```

This test attempts to understand (or characterize) the "Text Statement" generation for a customer given a children’s movie rented for 3 days. Because we do not understand the system (at least as of now), we expect the statement to be blank or contain any value that we are not aware of.
Let’s run the test and let it fail. When it does, *we have found out what the code actually does under that condition*.

```java
    java.lang.AssertionError:
    Expected :""
    Actual   :Rental Record for John, Total amount owed = 12.5. You earned 4 frequent renter points.</pre></div>
```
Now, that we know the behavior of the code, we can go ahead and change the test.

```java
import static com.code.legacy.movie.MovieType.CHILDREN;
import static org.junit.Assert.assertEquals;

public void shouldGenerateTextStatement(){
    String expectedStatement = "Rental Record for John, Total amount  owed = 12.5. You earned 4 frequent renter points";
    Customer john          = new Customer("John");
    Movie    childrenMovie = new Movie("Toy Story", CHILDREN);   
    int      daysRented    = 3;
    Rental   rental        = new Rental(childrenMovie, daysRented);
    john.addRental(rental);
    
    Sting statement = john.generateTextStatement();
    assertEquals(expectedStatement, statement);
}
```

**Hold on**, did we just copy the output generated by the code and placed into our test. *Yes, that is exactly what we did.*

We aren’t trying to find bugs right now. We are trying to put in a mechanism to find bugs later, bugs that show up as differences from the system’s current behavior. When we adopt this perspective, our view of tests is different: They don’t have any moral authority; they just sit there *documenting what the system really does*. At this stage, it’s very important to have the knowledge of what the system actually does.

**Q**uestion> What is the total number of tests that we write to characterize a system?

**A**nswer> It’s infinite. We could dedicate a good portion of our lives for writing case after case for any class in a legacy code.

**Q**uestion> When do we stop then? Is there any way of knowing which cases are more important than others?

**A**nswer> Look at the code we are characterizing. The code itself can give us ideas about what it does, and if we have questions, tests are an ideal way of asking them. At that point, write a test or tests that cover good enough portion of the code.

**Q**uestion> Does that cover everything in the code?

**A**nswer> It might not. But then we do the next step. We think about the changes that we want to make in the code and try to figure out whether the tests that we have will sense the problems that can happen. If they won’t, we add more tests until we feel confident that they will.

### How Much To Refactor
There is so much to refactor in legacy code, and we can not refactor everything. In order to answer this we need to go back to understanding our purpose of refactoring the legacy code.
We want to refactor legacy code to leave it cleaner than what it was when it came to us and to make it understandable for others.
With that said, we want to make the system better keeping the focus on the task. We don’t want to go crazy with refactoring trying to improve the whole system in a few days. What we want to do is *refactor the code that comes in our way of implementing any new change*. We will try and understand this better with an example in the next article.

{{< youtube 0U83rST3ang >}}

Let's move on to the [next article](/blog/lets_deal_with_legacy_code/) that explains how to deal with legacy code.

### References
- Working Effectively with Legacy code

