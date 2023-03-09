---
author: "Sarthak Makhija"
title: "Let’s deal with Legacy code"
date: 2018-04-12
description: "This article is in continuation with the previous article \"Let’s define Legacy code\" where we defined Legacy code. Let's understand \"Broken Window Theory\" and add a new feature to Legacy code."
tags: ["Legacy code", "Broken Window Theory", "Refactoring"]
thumbnail: /legacy-code.jpg
caption: "Photo by Markus Spiske on Pexels"
---
This article is in continuation with the <a href="/blog/lets_define_legacy_code/">previous article</a> where we defined some key aspects of Legacy code. In this article we will learn to deal with Legacy code.
Before we begin with an example, let’s take a moment to understand <em class="markup--em markup--p-em">Broken Window Theory.</em></p>

<img class="align-center" title="Broken Window Theory" src="/broken-window.jpeg" alt="Broken Window Theory" />

### Broken Window Theory
An academic theory proposed by *James Q. Wilson and George Kelling* in 1982 that used broken windows as a metaphor for disorder within neighbourhoods.

> One broken window if left unrepaired for a substantial amount of time, instills a sense of abandonment. So another window gets broken. People start littering. Graffiti appears. Serious structural damage begins. In a relatively short time, the building becomes damaged beyond the owner’s desire to fix it, and the sense of abandonment becomes reality.

Let’s not abandon our code, let’s repair the code as soon as we get an opportunity to repair it and let’s not get ourselves into a situation where damage is beyond our capacity to fix. Time to see this theory in action.

### Problem Definition Overview
The below code belongs to a hypothetical application "Movie Rental" that allows its customers to rent either Regular or Children’s movies for fixed number of days. The application also allows generation of a statement that the business calls "Text Statement". This application is running on the production environment for a long time without any issues and has become very popular. Now the business wants to generate an "HTML statement" without changing the logic for the amount computation.

```java
public class Customer {
    private String name;
    private List<Rental> rentals = new ArrayList<>();

    public Customer(String name) {
        this.name = name;
    }
    public void addRental(Rental arg) {
        rentals.add(arg);
    }
    public String getName() {
        return name;
    }
    public String statement() {
        double totalAmount = 0;
        String result = "Rental Record for " + getName() + "\n";
        for (Rental each : rentals) {
            double thisAmount = 0;

            //determine the amounts for each line
            switch (each.getMovie().getPriceCode()) {
                case Movie.REGULAR:
                    thisAmount += 2;
                    if (each.getDaysRented() < 2)
                        thisAmount += (each.getDaysRented() - 2) * 1.5;
                break;
                case Movie.CHILDRENS:
                    thisAmount += 1.5;
                    if (each.getDaysRented() < 3)
                        thisAmount += (each.getDaysRented() - 3) * 1.5;
                break;
            }
            //show figures for this Rental
            result += "\t" + each.getMovie().getTitle() + "\t" +
                    String.valueOf(thisAmount) + "\n";
            totalAmount += thisAmount;
        }
        //add footer lines result
        result += "Amount owed is " + String.valueOf(totalAmount) + "\n";
        return result;
    }
}

public class Movie {
    public static final int CHILDRENS = 2;
    public static final int REGULAR = 0;

    private String title;
    private int priceCode;

    public Movie(String title, int priceCode) {
        this.title = title;
        this.priceCode = priceCode;
    }
    //getters ignored
}

public class Rental {
    private int daysRented;
    private Movie movie;

    public Rental(Movie movie, int daysRented){
        this.movie = movie;
        this.daysRented = daysRented;
    }
    //getters ignored
}
```

The team decides to discuss different ways to handle this new requirement in legacy code.

{{< youtube aGGoW8YENKo >}}

And the team agrees to improve the code before implementing the new functionality. Scott and Jessica will be pairing on this. But, where do they start from? As mentioned in their discussion, they need to understand the code first, so they decide to write [Characterization Test(s)](/blog/lets_define_legacy_code#which-tests-to-write).

### First Characterization Test
Scott> How many tests should we write?

Jessica> Let’s look at the code. It should give us some hints.

Scott> I get it. We need a few rentals consisting of Regular and Children’s movie and the number of days rented should be greater than 2 or 3 to ensure the conditional logic based on `daysRented` gets executed. So, one test should cover a decent functionality.

Jessica> I can’t agree more. So let’s write it then.</p>

```java
public class CustomerUnitTest {
    @Test
    public void shouldGenerateStatement(){
        Customer john      = new Customer("John");
        Movie    regular   = new Movie("Black Panther", REGULAR);
        Movie    children  = new Movie("Lion King",     CHILDRENS);
        Rental rental1     = new Rental(regular, 3);  
        Rental rental2     = new Rental(children, 4);
        john.addRental(rental1);
        john.addRental(rental2);

        String statement = john.statement();
        assertEquals("", statement);
    }
}
```

Scott> Let’s run this and see it fail

```java
org.junit.ComparisonFailure:
Expected: ""
Actual:
 Rental Record for John
 Black Panther 3.5
 Lion King 3.0
 Amount owed is 6.5
```
Jessica> Great. We have made some progress. Let’s correct our test.

```java
public class CustomerUnitTest {
    @Test
    public void shouldGenerateStatement(){
        String expected = "Rental Record for John\n" +
                "\tBlack Panther\t3.5\n" +
                "\tLion King\t3.0\n" +
                "Amount owed is 6.5\n";

        Customer john      = new Customer("John");
        Movie    regular   = new Movie("Black Panther", REGULAR);
        Movie    children  = new Movie("Lion King",     CHILDRENS);
        Rental rental1     = new Rental(regular, 3);
        Rental rental2     = new Rental(children, 4);
        john.addRental(rental1);
        john.addRental(rental2);
        
        String statement = john.statement();
        assertEquals(expected, statement);
    }
}
```

*Jessica and Scott agree to write one test case covering a decent portion of the code. If this gives us confidence, we can live with one test for now else we can write a few more or include movies with `daysRented < 2`*.

Scott> Jessica, what type of test should a Characterization test be? Unit, Functional, Integration?

Jessica> Scott, it is not always possible to write unit or functional tests for legacy code. You might end up writing an *integration test* to begin with because you just want to know what system does. But, as soon as you get an opportunity, try to get your tests closer to the code.

Scott> Sure Jessica, let’s start the fun part. Let’s fix a broken window.

### Refactoring

Scott> Where do we start from?

Jessica> I believe the `statement()` method is a long method. We should try and make it a little smaller.

Scott> Agreed.

*Jessica and Scott agreed that `statement()` method is a long method. But, this agreement was not based on the number of lines in the method. It was based on how easy it is to comprehend the method or is a method doing more than one thing at a time, or it can be decomposed further.*

```java
public String statement() {
    double totalAmount = 0;
    String result = "Rental Record for " + getName() + "\n";
    for (Rental each : Rentals) {
        //determine amounts for each line
        double thisAmount = amount(each);

        //show figures for this Rental
        result += "\t" + each.getMovie().getTitle() + "\t" + String.valueOf(thisAmount) + "\n";
        totalAmount += thisAmount;
    }
    //add footer lines result
    result += "Amount owed is " + String.valueOf(totalAmount) + "\n";
    return result;
}

private double amount(Rental each) {
    double thisAmount = 0.0;
    switch (each.getMovie().getPriceCode()) {
        case Movie.REGULAR:
            thisAmount += 2;
            if (each.getDaysRented() < 2)
                thisAmount += (each.getDaysRented() - 2) * 1.5;
        break;
        case Movie.CHILDRENS:
            thisAmount += 1.5;
            if (each.getDaysRented() < 3)
                thisAmount += (each.getDaysRented() - 3) * 1.5;
        break;
    }
    return thisAmount;
}
```

Jessica> The `switch` statement has gone out and the extracted `amount()` method does one thing which is getting the amount for a given rental.

Scott> Let’s continue refactoring. I am in a mood to clean up everything.

Jessica> Hold on Scott, we need to run tests before we move on.

*And the test ran successfully.*

*While working with Legacy code it is important to take smaller steps and follow the refactoring cycle. Refactor -> Run Tests -> Refactor*

Scott> Sure. Jessica, are we in a position to remove the comment "determine the amounts for each line" from previous code?

Jessica&gt; Yes, we can remove it.</p>

```java
public String statement() {
    double totalAmount = 0;
    String result = "Rental Record for " + getName() + "\n";
    for (Rental rental : Rentals) {
        double thisAmount = amount(rental);

        //show figures for this Rental
        result += "\t" + rental.getMovie().getTitle() + "\t" + String.valueOf(thisAmount) + "\n";
        totalAmount += thisAmount;
    }
    //add footer lines result
    result += "Amount owed is " + String.valueOf(totalAmount) + "\n";
    return result;
}

private double amount(Rental rental) {
    double thisAmount = 0.0;
    switch (rental.getMovie().getPriceCode()) {
        case Movie.REGULAR:
            thisAmount += 2;
            if (rental.getDaysRented() < 2)
                thisAmount += (rental.getDaysRented() - 2) * 1.5;
        break;
        case Movie.CHILDRENS:
            thisAmount += 1.5;
            if (rental.getDaysRented() < 3)
                thisAmount += (rental.getDaysRented() - 3) * 1.5;
        break;
    }
    return thisAmount;
}
```
*Remove comments from legacy code when you have captured their complete essence. Though I did take some liberty to rename variable along with removing the comment, it is always ideal to take smaller steps when you are beginning to understand legacy code. As you grow in confidence, you might want to take bigger steps but one test failure and the reality reveals itself.*

Scott> Let’s look at the `amount()` method. It depends on the `priceCode` from `movie` but is placed in the `Customer` object. We should move this method to the place where it belongs.

Jessica> Yes, let’s do a few method movements (in the interest of this article).

```java
//Customer
public String statement() {
    double totalAmount = 0;
    String result = "Rental Record for " + getName() + "\n";
    for (Rental rental : Rentals) {
        double thisAmount = rental.amount();

        //show figures for this Rental
        result += "\t" + rental.movieTitle() + "\t" + String.valueOf(thisAmount) + "\n";
        totalAmount += thisAmount;
    }
    //add footer lines result
    result += "Amount owed is " + String.valueOf(totalAmount) + "\n";
    return result;
}

//Rental
double amount() {
    return movie.amount(this.daysRented);
}

//Movie
double amount(int daysRented) {
    double thisAmount = 0.0;
    switch (this.getPriceCode()) {
        case Movie.REGULAR:
            thisAmount += 2;
            if (daysRented < 2)
                thisAmount += (daysRented - 2) * 1.5;
        break;
        case Movie.CHILDRENS:
            thisAmount += 1.5;
            if (daysRented < 3)
                thisAmount += (daysRented - 3) * 1.5;
        break;
    }
    return thisAmount;
}
```
*I did a few movements. Moved the `amount()` method to the `Rental` object and then to the `Movie` object and ran the tests. It should be noted that this is our first opportunity to write unit tests for `Rental` and `Movie`. I won’t, for this article, but I assume you will*.

Scott> Jessica, I have a question. Movie has a switch statement based on different types of movies. Shall we introduce some polymorphism here?

Jessica> I don’t think it is coming in our way of implementing the HTML statement functionality.

> Scott has raised a valid point, but we need to remember one thing, “we refactor the code that comes in our way”. At this point, we need to implement the HTML statement and the switch code does not come in the way of our new feature, neither do the magic numbers 2 nor 1.5. If you want to continue with small refactorings that are not coming in your way, say changing Magic Numbers to Constants, go ahead and do it but do not move away from your actual task that is implementing the HTML statement.

Scott> I get that. Thank you. The `statement()` method in the `Customer` is short enough. Shall we pause our refactoring here?

Jessica> We could, but one thing that is bothering me is this method seems to be generating 3 parts of the statement, and I can see it clearly: header, body and footer. *If the effort is not huge* we should try and extract this code into different methods.

Scott> You clearly have an eye for refactoring. Let’s do it.</p>

```java
public String textStatement() {
    return textHeader() + textBody() + textFooter();
}

private String textHeader() {
    return "Rental Record for " + getName() + "\n";
}

private String textBody() {
    String result = "";
    for (Rental rental : Rentals) {
        result += "\t" + rental.movieTitle() + "\t" + String.valueOf(rental.amount()) + "\n";
    }
    return result;
}

private String textFooter() {
    return "Amount owed is " + String.valueOf(totalAmount()) + "\n";
}

private double totalAmount() {
    double totalAmount = 0.0;
    for (Rental rental : Rentals) {
        totalAmount += rental.amount();
    }
    return totalAmount;
}
```
*I cheated again. Did lot more than what I should have done, renamed methods to be **text\***, duplicated for loops (over rentals) to calculate the total amount, repeated the same in the `textBody()` method*.

Is that justified? Well, how many rentals do we expect to have for a customer? What is the cost of iterating over them twice? If it is not significant, go ahead and use it. What does it give me? Look at the `statement()` (renamed as `textStatement()`) method now.

Jessica> Now, we are done with refactoring. We can introduce the HTML statement functionality now.

### Conclusion

Jessica and Scott went on to implement the HTML functionality (with tests) and they did a lot to clean up the existing code. The code is much more understandable that it used to be.

They might not have cleaned up everything, but they have left a great deal of understanding trace for others to follow.

They followed *Cover and Modify*, *Boy Scout rule*, *Refactoring cycle* and refactored enough to finish the new functionality, in-short dealt with Legacy code professionally.

### References

- Refactoring Improving The Design Of Existing code
- [Refactoring Catalog](https://refactoring.com/catalog)