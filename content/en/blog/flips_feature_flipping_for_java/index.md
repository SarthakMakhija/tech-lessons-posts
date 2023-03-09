---
author: "Sarthak Makhija"
title: "Flips: Feature Flipping for Java"
date: 2017-10-07
description: "Flips is an implementation of the Feature Toggles pattern for Java and Spring (Spring Core / Spring MVC/ Spring Boot) based application. Feature Toggle is a powerful technique that allows teams to modify the system behavior and deliver new functionality to end-users rapidly and safely. Let's understand what \"Flips\" is all about"
tags: ["Flips", "Feature toggles", "Spring", "Spring MVC"]
thumbnail: /flips-feature-title-image.png
caption: "Background by Linus Nylund on Unsplash"
---

<img title="Flips" src="/flips.jpg" alt="Flips" />

[Flips](https://github.com/Feature-Flip/flips) is an implementation of the Feature Toggles pattern for Java and Spring (Spring Core / Spring MVC/ Spring Boot) based application.
[Feature Toggle](https://martinfowler.com/articles/feature-toggles.html) is a powerful technique that allows teams to modify the system behavior and deliver new functionality to end-users rapidly and safely.

### Why Another Library for Feature Toggles?
The idea behind **Flips** is to let the clients implement toggles with *minimum configuration and coding*.

The main motivations behind implementing this library were:
- Should be simple to use
- Should require minimal configuration and code
- Should be able to flip a feature based on various conditions
- Should be able to flip a feature based on a combination of different conditions
- Should be possible for the clients to create custom conditions to suit their requirements

The library **Flips** works with Java 8 and Spring Core/Spring MVC/Spring Boot, and is available for web and non-web applications.

### What Does Flips Offer?
The library **Flips** provides various conditions to flip a feature. The image below summarizes the features:
<img src="/flips-features.png"/>

Any feature can be flipped ON or OFF based on different conditions that can be value of a property, current active profiles, days of the week, or a combination of these, etc.
Let’s get started with in-depth understanding of these features.

### Getting Started
Include the necessary dependency:
```gradle
<dependency>
   <groupId>com.github.feature-flip</groupId>
   <artifactId>flips-web</artifactId>
   <version>1.0.1</version>
</dependency>
```
Or,

```gradle
<dependency>
  <groupId>com.github.feature-flip</groupId>
  <artifactId>flips-core</artifactId>
  <version>1.0.1</version>
</dependency>
```

### Detailed Description of all the annotations
The library **Flips** provides various annotations to flip a feature. Let’s have a detailed walk-through of all the annotations:

**@FlipOnEnvironmentProperty**

`@FlipOnEnvironmentProperty` is used to flip a feature based on the value of an environment property.</p>

```java
@Component
class EmailSender {
    @FlipOnEnvironmentProperty(property = "feature.send.email", 
                               expectedValue = "true")
    public void sendEmail(EmailMessage emailMessage) {
    }
}
```

**@FlipOnProfiles**

`@FlipOnProfiles` is used to flip a feature based on the environment in which the application is running.

```java
@Component
class EmailSender {
    @FlipOnProfiles(activeProfiles = {"dev", "qa"})
    public void sendEmail(EmailMessage emailMessage) {
    }
}
```
        
The feature `sendEmail` is enabled if the current profile (or the environment) is either *dev* or *qa*.

**@FlipOnDaysOfWeek**

`@FlipOnDaysOfWeek` is used to flip a feature based on the day of the week.

```java
@Component
class EmailSender {
    @FlipOnDaysOfWeek(daysOfWeek = {DayOfWeek.MONDAY})
    public void sendEmail(EmailMessage emailMessage) {
    }
}
```
The feature `sendEmail` is enabled if the current day is MONDAY.

**@FlipOnDateTime**

`@FlipOnDateTime` is used to flip a feature based on date and time.

```java
@Component
class EmailSender {
    @FlipOnDateTime(cutoffDateTimeProperty = "default.date.enabled")
    public void sendEmail(EmailMessage emailMessage) {
    }
}
```
The feature `sendEmail` is enabled if the current datetime is equal to or greater than the value (in `ISO-8601` format) defined by the `default.date.enabled` property.

**@FlipOnSpringExpression**

`@FlipOnSpringExpression` is used to flip a feature based on the evaluation of a Spring expression.

```java
@Component
class EmailSender {
    @FlipOnSpringExpression(expression = "T(java.lang.Math).sqrt(4) * 100.0 
                                          < T(java.lang.Math).sqrt(4) * 10.0")
    public void sendEmail(EmailMessage emailMessage) {
    }
}
```
The feature `sendEmail` is enabled if the expression evaluates to TRUE. This annotation happens to be one of the most powerful annotations in the Flips library. Why so?
One could always write a custom spring component and use the same in `@FlipOnSpringExpression` to flip a feature.

**@FlipBean**

`@FlipBean` is used to flip the invocation of a method with another method defined in a different bean.

```java
@Component
class EmailSender {
    @FlipBean(with = SendGridEmailSender.class)
    @FlipOnProfiles(activeProfiles = "DEV")
    public void sendEmail(EmailMessage emailMessage) {
    }
}
```
This will flip the invocation of the `sendEmail` method with a method (having same signature) defined in the `SendGridEmailSender` Spring component if the current profile is DEV.

**@FlipOff**

`@FlipOff` is used to flip a feature off.</p>

```java
@Component
class EmailSender {
    @FlipOff
    public void sendEmail(EmailMessage emailMessage) {
    }
}
```

The feature `sendEmail` is always DISABLED.

**Combining annotations**

```java
@Component
class EmailSender {
    @FlipOnProfiles(activeProfiles = "dev")
    @FlipOnDaysOfWeek(daysOfWeek={DayOfWeek.MONDAY})
    public void sendEmail(EmailMessage emailMessage) {
    }
}
```
The feature `sendEmail` is enabled if the current profile is dev **AND** the current day of the week is MONDAY.

### Import Flip Context Configuration
To bring all Flips-related annotations into effect, one needs to import `FlipContextConfiguration` or `FlipWebContextConfiguration`.

```java
@SpringBootApplication
@Import(FlipWebContextConfiguration.class)
class ApplicationConfig {
    public static void main(String[] args) {
        SpringApplication.run(ApplicationConfig.class, args);
    }
}
```
Please refer to the sample application config [here](https://github.com/SarthakMakhija/flips-samples/blob/master/flips-sample-spring-boot/src/main/java/com/finder/article/ApplicationConfig.java).

### Creating Custom Annotations
All the annotations provided by the library are of type `@FlipOnOff`, that is essentially a meta-annotation. So, it is possible to create a custom annotation annotated with `@FlipOnOff` at the method level:
```java
@Target({ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME) 
@FlipOnOff(value = MyCustomCondition.class) !!Important
public @interface MyCustomAnnotation {
}
```

As a part of this annotation, specify the custom condition that will evaluate this annotation.

```java
@Component
public class MyCustomCondition implements FlipCondition {
    @Override
    public boolean evaluateCondition(FeatureContext fContext,
                                     FlipAnnotationAttributes attributes){

        //code to evaluate flip condition
        return false;
    }
}
```
This Condition class needs to implement `FlipCondition` and MUST be a Spring Component.
That is it! You can use your custom annotation to any method to flip it ON or OFF based on your condition.

### What is FeatureNotEnabledException
The exception `FeatureNotEnabledException` is thrown if a disabled feature is invoked. In case of a web application, one could the use [flips-web](https://mvnrepository.com/artifact/com.github.feature-flip/flips-web/1.0.1) dependency, that also provides a `ControllerAdvice` to handle this exception.
It returns a default response and a status code of 501, which can be overridden. Please refer to the [sample project](https://github.com/SarthakMakhija/flips-samples/tree/master/flips-sample-spring-boot/src/main/java/com/finder/article/advice) for more information.

### Wrap Up
We believe the MVP is done and features like flipping at runtime and supporting database-driven feature flips are in the pipeline.
For any custom flip condition, one could go ahead and use `@FlipOnSpringExpression` with your custom spring bean to determine the flip condition.
If you want to have a look at the code or even want to contribute, you can check out [Flips](https://github.com/Feature-Flip/flips). 

Feel free to share any feedback.
