---
author: "Sarthak Makhija"
title: "Code without automated tests? Are we serious?"
date: 2023-02-24
description: "What would it really mean to write code without automated tests, to deliver software without automated tests. Why would someone even think of writing code today and adding tests later?"
tags: ["Agile", "Testing", "Refactoring"]
thumbnail: /code-without-tests.png
caption: "Photo by Ana Arantes on Pexels"
---

Automated tests are an essential part of every piece of code that we write. The benefits of these tests are so compelling that it does not even
make sense to think about writing code without tests or writing code today and adding tests later. Despite the benefits, we still see code without tests,
we still see ideas like "writing code today and adding tests when the delivery pressure reduces" floating around.

I don't have real reasons as to why such ideas would float around, but I can speculate.

**Speculation (1)**
Adding tests takes a lot of time

**Speculation (2)**
There is a lot of delivery pressure today and there is a firm belief that tomorrow will be better and that is when we will go and add tests

What I want to do as a part of this article is try and understand how would our world look like with such ideas and does it even make sense to consider them.
With that said, I will keep the idea of TDD aside for this article.

<blockquote class="wp-block-quote">
    <p>What would it really mean to write code without automated tests, to deliver software without automated tests. Why would someone even think of writing code today and adding tests later?</p>
</blockquote>

### Does it really take time to add tests?

One of the possible arguments around not adding automated tests could be the "time".

<blockquote class="wp-block-quote">
    <p>Argument1: It takes time to add tests and time is really costly. We need to finish our story and there is this delivery pressure.</p>
</blockquote>

Let's see how fair is that argument by considering a method ```leftShift``` which left shifts the elements of a slice by 1, let's assume non-empty slice for now.

```golang
type slice struct {
  elements []int
}

func newSlice(elements []int) *slice {
  return &slice{elements: elements}
}

func (s *slice) leftShift() {
    for index := 0; index < len(s.elements)-1; index++ {
      s.elements[index] = s.elements[index+1]
    }
    s.elements[len(s.elements)-1] = 0
}
```

<div class="align-center">
    <img src="/left-shift-slice.png"/>
</div>

What this code does is pretty simple -
- drops the element at index 0
- moves each element to its left
- puts a zero at the last index

Let's add a unit test for the same.

```golang
package main

import (
  "reflect"
  "testing"
)

func TestLeftShiftsANotEmptySliceBy1(t *testing.T) {
    slice := newSlice([]int{10, 20, 30, 40, 50})
    slice.leftShift()

	expected := []int{20, 30, 40, 50, 0}

	if !reflect.DeepEqual(expected, slice.elements) {
		t.Fatalf("Expected %v, received %v after performing left shift",
			expected,
			slice.elements,
		)
	}
}
```

As a part of this test, we perform a left shift on a slice and assert the elements against the expected after shift operation. That's really it.

It only requires us to understand how to write unit tests.
Honestly, it doesn't take a lot of time to add tests, be it unit tests, integration tests, contract tests or API tests, once we understand a few things including -
- What do these tests stand for
    - It is essential to understand what is "unit" in a unit test, what is an "integration" test etc
- What purpose do these tests serve
    - It is essential to answer questions like "why can't we have all integration tests and zero unit tests"
- And, how to write these tests
    - It is essential to answer questions like "how do I write tests in X programming language with Y framework"

Once we answer all these questions, it doesn't take a lot of time to add tests.

There is still a forked argument that I can think of.

<blockquote class="wp-block-quote">
    <p>Argument2: What we build is not as simple as left shifting a slice, our systems are complex and fancy. We can't just add one test and be done. Hence, adding so many tests would take time, which of course we don't have.</p>
</blockquote>

The answer to this lies in the argument itself. If left shifting a slice needs an automated test, any fancy and complex system would need them too.

And yes, your system is not as simple as left shifting some elements but at the same time, the fancy system is not built in a day. It is built piece by piece
gradually, so why not add tests for every small piece that gets built.

*Important Side note:* If we remove the assumption that our slice is non-empty (ie; ```elements``` within the ```slice``` struct is non-empty), ```leftSlice``` method will fail.
In fact, at this point in time, the only way to conclude that an empty slice will result in a failure is by walking through the code. Once we have the test
for the same, not only does it give us a safety net but also serves as a **live documentation** which gets updated everytime the behavior of the changes.

### Code today and add tests later

One of the other theories that I have heard is "let's code today and add tests tomorrow or maybe later". There could be multiple reasons for this theory
but probably, "delivery pressure", and a beautiful belief "code without automated tests is ok" should be the main reasons for this wonderful idea to pop up.

Let's see what would happen if we write code today and add tests later, forget TDD. Consider our favorite method ```leftShift``` and assume -
- no tests are written
- a week has passed by, and now we are adding tests

Let's look at various challenges that would come up -

**Boring**

I don't have a better word for this stuff. We need to look at the code and understand what it does. Once that understanding is built, we need to write tests.
One might argue, "Why build an understanding", just write "characterization tests".
Sure, but how would you figure out corner cases, you need some background to think about corner cases.

This stuff just seems boring to me, looking at the code, figuring out various cases that too after 7 days, and adding tests.

**Possibility of missing corner cases**

There is a good possibility that we will miss corner cases if we decide to add tests later because the cases, or the context is not fresh anymore.

**Lack of motivation**

What would be the motivation to add tests 7 days later? Lack of coverage in Sonar?

Well, someone might even say this - "the code is already working somewhere (in Dev, QA or even in prod), why bother adding tests now".
It is so easy for such a thought process to set in and once it sets in, tests would only be added for finishing some formality.

**Delayed refactoring**

In the absence of tests, refactoring for various parts of code will get delayed. This in turn has a huge drawback, a method which is 50 lines today might
grow to 100 lines in 7 days. Not only are the number of code smells going to increase later, but even refactoring might become tricky or may take longer.

What are we basing theory of "adding code today and tests later" on? How is tomorrow going to be any different? Is sun going to shine too brightly tomorrow?
Are we going to stop churning stories tomorrow? Are we going to just add tests and do nothing else?

The overall idea is just flawed.

I know I could have hurt your emotions, but let's take a look at some benefits of automated tests and see the real gains.

### Benefits of automated tests

Automated tests provide a lot of benefits. I will list a few -

**Provide confidence**

Automated tests are like a certification for a working code. I know ```leftShift``` is working properly everytime its tests pass.
In fact, automated tests act as a mapping between questions and answers. I can ask various questions to unit tests -
- How would ```leftShift``` behave if I pass a slice with empty elements
- How would ```leftShift``` behave if I pass a slice with a single element
- How would ```leftShift``` behave if I pass a slice with N elements, where N > 1
- How would ```leftShift``` behave if I pass a slice with N elements containing duplicates, where N > 1
- How would ```leftShift``` behave if I invoke the method with a nil receiver, ie; ```s``` of ```(s *slice)``` is nil

It is a huge confidence booster :) if all these questions are answered by passing tests.

**Act as safety net**

Automated tests are a brilliant safety net, I can go ahead and refactor code without any fear. I know I have tests which would fail loudly if I mess things up, so there is no fear of making mistakes while refactoring.

I think it would be a very courageous move to refactor code without tests. (*Honestly, I don't know if it is a courageous move or a stupid move.*)
But, if I decide to refactor code without tests, I think I would be blocked by anxiety, there will be a constant banging in the head - what if refactoring breaks the code,
can I just stop refactoring here, is it really necessary to refactor etc. With tests written for the code, there is no case of anxiety or fear.

<blockquote class="wp-block-quote">
	Tests are anxiety busters :), especially unit tests
</blockquote>

**Provide quick feedback**

Automated tests provide quick feedback on any change that is done in the code.
Assume (just assume) we are refactoring a long method, and we do not have tests. Let's try to imagine what the world would look like now -
1. Extract a piece of code into a new method
2. Run the entire application, send some requests and see if the extraction worked
3. It worked, congratulations
4. Rename the extracted method
5. Run the entire application, send some requests and see if renaming worked
6. It worked, congratulations again
7. Change the number of parameters of the extracted method
8. Run the entire application, send some requests and see if the change in number of parameters worked
9. You really deserve congratulations
10. The method is feature envy, let's move it to the right class

...

If there were unit tests, we could have run them every time on every change, and it would have been way quicker than running the entire application N times.
There is a strange part that I don't seem to understand, I will explain.
If the argument for not writing tests or deferring writing tests is "lack of time", then, where would you get time for running the application N times
to validate if a change has worked.

**Provide ability to move fast under pressure**

Automated tests provide the ability to move fast under pressure. Let's say a production defect is found and  if we have unit tests, we are SAFE.
Under production issue pressure what we don't want to do is make a small problem explode into a bigger one.
With unit tests in place, all we do is -
- replicate the issue by adding a failing test
- make the necessary code changes to pass the test
- run the entire test suite, if all is "green", we are good

*Important Side note:* What we also don't want to do is "just write UI tests in a web application". We need quick feedback for any changes that we make in the code, and it becomes even more essential to figure out what kind of tests would make more sense for a given situation.

For instance, if there is a method which sorts all the "orders" based on "order date". It does not make sense to test it as a part of some UI based test,
all this method does is "sorting of a collection based on an attribute". Just add unit tests for the use-case and that is good enough to prove that a part
of the functionality (sorting) is working fine.

**Act as documentation**

Automated tests act as documentation for what the code does.
I don't need to go through ```leftShift``` method to understand its behavior -
- what if it is invoked with a slice containing empty elements
- what if it is invoked with a slice containing just one element

I will just go and look at the tests. In fact, adding tests is one way of creating a trail of understanding for the readers. Readers of the code don't need to make too much of an effort to find such answers,
they can directly look at the tests and find *most* of the answers.

**Provide an opportunity to think from a client's perspective**

Automated tests give an opportunity to think from a client's perspective and provide a lot of opportunities to improve the API.

Let's imagine a struct ```LinkedList``` and I am required to add a behavior which
rotates a linked list left by N. (Assume, I am not doing TDD). I start with a behavior called ```rotateList(n int)```, build it and now I go on to adding a test.
Test would look something like this -

```golang
func TestRotatesALinkedListBy1(t *testing.T) {
    linkedList := LinkedList{}
    linkedList.rotateList(1)
    ....
}
```

One of the first things that I notice is the name of the method ```rotateList```.
In the expression ```linkedList.rotateList```, do I need to call the behavior as ```rotateList``` or just ```rotate``` is good enough
because it is invoked on a list. I would go and rename it to ```rotate``` and now my client call looks like ```linkedList.rotate(1)```.

Once I start paying attention to the entire expression ```linkedList.rotate(1)```, I would realize ```rotate(1)``` is not making sense. It does not tell the clients or the readers that the intention is to
rotate list by 1. What if it were renamed to ```rotateBy()```, it would make my client call look like this ```linkedList.rotateBy(1)```

Now, probably the last thing would be, ```linkedList.rotateBy(1)``` does not tell the clients about the direction of rotation.
What if it were renamed to ```rotateLeftBy()```, it would make my client call look like this ```linkedList.rotateLeftBy(1)```

```golang
func TestRotatesALinkedListLeftBy1(t *testing.T) {
    linkedList := LinkedList{}
    linkedList.rotateLeftBy(1)
    ....
}
```

<blockquote class="wp-block-quote">
	Tests actually influence the design of class - at micro & macro levels. And actually help write more intuitive code (because they are written from the outside-in perspective, it makes for more readable code... than code which is influenced by "implementation details").
</blockquote>

By not adding automated tests or by deferring addition of tests, we are just losing all these advantages.

### Fundamental idea

One of the fundamental ideas behind coding is to break a problem into small pieces say tasks and each of these tasks requires a shifting episode between
code and tests. I will explain.  Let's say we want to build a [linked list](https://www.geeksforgeeks.org/linked-list-set-1-introduction/) which supports ```get(key)```
and ```put(key, value []byte)```.

The idea would be to break the problem into small solvable problems and attempt to solve the simplest. One of the task lists could be -
- Build a linked list which supports ```put(key, value)``` with a single node
- Add tests to assert that ```put(key, value)``` works
- Enhance linked list which now supports ```put(key, value)``` with multiple nodes
- Add tests to assert that ```put(key, value)``` works
- Add ```get(key)``` in linked list
- Add tests to assert that ```get(key)``` works if key is found
- Add tests to assert that ```get(key)``` works if key is not found; if it does not work, go back and change the code
- Add tests to assert that ```get(key)``` works even if list is empty; if it does not work, go back and change the code
- Looks good, relook at all the test cases and see if we have missed anything

It looks like there is a continuous dialogue between code and tests, both of them seem to be talking to each other.

A test says, "Hey, ```get``` is failing when a key is not present", code says, "Updated, please run yourself again".

Code says, "I will return an error if value for a key is not present" and test would say "let me check if it makes sense from a client's perspective and get back".

Essentially, it's the very act of design and coding that needs small focused episodes with actual code, driving the code with various scenarios.
Tests and TDD help by adding some specific guidance to it, which is a huge benefit.

*If all of this is not good enough, then ..*

### Would you buy a car without brakes?

One of the things that the automated tests provide is a "safety net" which in turn allows us to make changes in code with confidence. I am not sure why do we even call software delivery a delivery, without automated tests.
Let's try to draw an analogy between "building software" and "manufacturing car".

<blockquote class="wp-block-quote">
    <p>If it is ok to write code without tests, to deliver software without tests, then we should be ok to buy a car without brakes. </p>
</blockquote>

How would it feel if a car manufacturer told us to buy a car without brakes. If adding tests in the code can be deferred or be considered non-essential, why can't brakes in a car be considered non-essential?
The point is why can't brakes be added later on if time permits, if there is no delivery pressure or, if there are less delivery orders?

We really need confidence in the car that we are driving, we also need confidence in the brakes, and if confidence is such an important thing, then
why not build confidence in the code by adding automated tests for every piece of code that we write?

*If our software is a car, then automated tests are the brakes.* Like we said, we need confidence in the brakes as well which essentially means, it is not just about writing
tests because of some code coverage policy in the organization, it is about "building a trail of understanding for all the readers", about stating "what is it that a piece of code does", "when does it fail", "what kind of inputs does it take" and so many other things.

<blockquote class="wp-block-quote">
	It is our responsibility to ensure these "brakes" exist in the code (or the system) and they are reliable. They should not exist just for the sake of existing. 
</blockquote>

### Time is still a factor, isn't it?

Yes, it is and that is where a combination of a couple of things is needed, **passion to learn and improve** and **the support from leadership in the
project (or the account).**

Let's see a few ways in which this can be done -

**Setting realistic expectations**

It is essential for the leadership in the account to be aware of the situation on the ground before making any commitment. If the team needs time to learn
and work on automated testing, it should be baked in the release plan, probably by keeping a low velocity in the initial iterations (assuming agile) and then gradually
attempting to increase it as the team becomes more and more comfortable with automated testing.

**Investing in the team**

It is essential to provide learning resources like books / videos / articles to the team to help them learn automated testing. At the same time, it is
very much needed for the team to come together and align on (automated) testing strategy.

**Building a culture of continuous improvement**

It is essential to build a culture of continuous improvement, "Done" is not an option. If a team is not doing automated testing, it needs to learn, probably
a few members need to act as "catalysts".
They shouldn't stop at learning, they need to practice it, check how various open source projects are doing it and teach it to other team members.
A team can try to do "lunch and learn", "mob pairing", or "collaborative code improvement" type of sessions.

*These are just a few ways, but the idea is - teams should make attempts to improve, and the necessary "support system" should be built around the team.*

### Conclusion

I don't see any reason for not writing tests or deferring the addition of tests. I think once you get addicted to "quick feedback", this point of developing
without tests, writing code today and adding tests later and not doing TDD automatically goes away.

If a team finds it difficult to write tests, or it takes too long to write tests, then the team needs to practice it more.
Practice till it becomes a habit. Not adding tests is not a solution. It is an easy hack but an expensive one. At the same time,
the necessary "support system" should be built around the team.

If a team believes their software has been working without issues and that too without tests, I think it is just a matter of "when", not "if". Try to
get better at things before it all comes crashing down.

If a team believes there is delivery pressure today and tests can be added tomorrow, then the team needs to be sure of one thing - "That tomorrow is never coming".

### Mentions
I would like to thank [Gurpreet Luthra](https://life-lessons.in/), [Unmesh Joshi](https://github.com/unmeshjoshi)
and [Sunit Parekh](https://www.sunitparekh.in/about/) for providing feedback on the article. Thank you Gurpreet, Unmesh and Sunit.