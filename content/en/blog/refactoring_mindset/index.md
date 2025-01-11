---
author: "Sarthak Makhija"
title: "Refactoring Mindset"
date: 2025-01-11
description: "
"
tags: ["Refactoring", "Mindset"]
thumbnail: "/talk_title.webp"
caption: "Photo by Pixabay on Pexels"
---

### a section or a para that this article covers existing codebase.

### Safety net

### Mindset

#### Small changes, small commits

Starting small is essential when refactoring. When you are starting to refactor, identify the smallest change that you can make. I think it is important to be 
patient, and not worry too much about the final state. Each incremental change contributes to improving the code step by step.

Take the `TaskList` example: 

```java
private final Map<String, List<Task>> tasks = new LinkedHashMap<>();
```

The variable `tasks` is a bad name (or a misleading name). It actually represents `Projects`, where every `Project` has a name and a collection of `Tasks`. At this stage, the `Project`
abstraction is buried within the `Map`. The smallest refactoring would be to rename `tasks` to `projects` without worrying too much about the domain abstraction
`Project`. After renaming, run all the tests to ensure nothing breaks and commit the change.

This small, low-risk refactoring instantly adds clarity to the code. For example, the following method:

```java
private void addProject(String name) {
    tasks.put(name, new ArrayList<>());
}
```

becomes 

```java
private void addProject(String name) {
    projects.put(name, new ArrayList<>());
}
```

With this change, the method name `addProject` aligns clearly with the collection `projects`, allowing the domain concept of 
`Project` to begin surfacing more clearly.

#### Recognize Code Smells

There are certain smells which are more common than the others. For example, the "bad name" smell is more common than "long method," 
and "long method" is more common than "refused bequest." Being familiar with these common smells makes refactoring significantly easier.

Let’s revisit our `TaskList` example:

```java
private void show() throws IOException {
    for (Map.Entry<String, List<com.codurance.training.tasks.Task>> project : projects.entrySet()) {
        writer.write(project.getKey());
        writer.write("\n");
        for (com.codurance.training.tasks.Task task : project.getValue()) {
            writer.write(String.format("[%c] %d: %s%n", 
                       (task.isDone() ? 'x' : ' '), task.getId(), task.getDescription())
            );
        }
    }
}
```

The method `show` is displaying (/showing) the details of all the tasks across all the projects. Let's see if we can identify some smells
in this method.

While the method isn’t long in terms of the number of lines, it could certainly be broken down further. A method like this could be considered 
a [long method](https://refactoring.guru/smells/long-method) because, even though the number of lines is small, it might be hiding other 
potential smells that are not clearly visible.

The show method directly accesses the private fields of the `Task` class, which violates encapsulation and leads to 
[inappropriate intimacy](https://refactoring.guru/smells/inappropriate-intimacy) between classes. This is problematic because a class 
should have the freedom to change the internal structure of its fields without affecting other parts of the code. In other words, 
a class should expose behaviors that operate on its internal data rather than allowing external access to its internal state. 
This aligns with the **Tell, Don’t Ask principle**: tell the object what to do, don’t ask for its internal state.

Additionally, if we find ourselves working with a [data class](https://refactoring.guru/smells/data-class) (a class that contains data but 
lacks meaningful behavior), the least we can do is restrict the visibility of its getters and setters to minimize access 
to its fields.

Let's start small again and extract `String.format` into a separate method.

```java
private void show() throws IOException {
    for (Map.Entry<String, List<com.codurance.training.tasks.Task>> project : projects.entrySet()) {
        writer.write(project.getKey());
        writer.write("\n");
        for (com.codurance.training.tasks.Task task : project.getValue()) {
            writer.write(format(task));
        }
    }
}

private static String format(Task task) {
    return String.format("[%c] %d: %s%n", 
                         (task.isDone() ? 'x' : ' '), task.getId(), task.getDescription()
           );
}
```

The `format` method is currently static, which means it doesn't belong to the class it's defined in since it doesn't use any of its fields. 
However, it makes use of the fields of the `Task` class, so it should be moved to the `Task` class itself. To do this, we can simply 
remove the `static` keyword and then, using a tool like `IntelliJ`, press `F6` to refactor the method and move it to the `Task` class.

The `Task` and `TaskList` look like the following:

```java
String format() {
    return String.format("[%c] %d: %s%n", (isDone() ? 'x' : ' '), getId(), getDescription());
}
```

```java
private void show() throws IOException {
    for (Map.Entry<String, List<com.codurance.training.tasks.Task>> project : projects.entrySet()) {
        writer.write(project.getKey());
        writer.write("\n");
        for (com.codurance.training.tasks.Task task : project.getValue()) {
            writer.write(task.format());
        }
    }
}
```

Run all the tests to ensure nothing breaks, add tests for the new `format` method inside `Task` and commit the change.
Since the `format` method is now part of the `Task` class, the `getDescription` method is no longer needed. Remove it, run the 
tests again, and commit this change as well.

This refactoring brings a positive side effect: the `Task` class, which was primarily a data class, is now beginning to contain 
meaningful behavior. As we continue, we’ll likely be able to eliminate other getter methods, further encapsulating the behavior 
within the class itself.

#### Avoid getting sidetracked

It’s important to stay focused on the task at hand during refactoring. However, the world is full of distractions :), making it easy to get 
sidetracked. In our `TaskList` example, it is tempting to consider introducing abstractions like `Project`, reworking the switch case, 
or even addressing components like `IdGenerator`. 

The previous refactoring focused on improving the `show` method. This refactoring started to improve the `Task` class by introducing the `format`
method. Thus, it's important to stay focused and choose one of the following directions:
- Continue improving the `show` method, or
- Explore if the `Task` class can be further improved, potentially removing the `getId` method

I’ve decided to stick with improving the `show` method. The [git commit history](https://github.com/SarthakMakhija/task-list-refactoring/commit/883ac3d4a34c2bef8ba1cbdc5d1fe879b1e7c808)
also shows the path that I took.

We can extract the inner for-loop `com.codurance.training.tasks.Task task : project.getValue()` in a separate method. The extracted `format`
method is shown below:

```java
private void show() throws IOException {
    for (Map.Entry<String, List<com.codurance.training.tasks.Task>> project : projects.entrySet()) {
        writer.write(project.getKey());
        writer.write("\n");
        List<Task> tasks = project.getValue();
        format(tasks);
    }
}

private void format(List<Task> tasks) throws IOException {
    for (Task task : tasks) {
        writer.write(task.format());
    }
}
```

The `format` method works on a `List<Task>` and uses the `writer` field from the `TaskList` class. If an instance of 
`java.io.Writer` were passed as a parameter to the `format` method, the method would no longer depend on the `TaskList` instance, 
making it `static`.

```java
private static void format(List<Task> tasks, Writer writer) throws IOException {
    for (Task task : tasks) {
        writer.write(task.format());
    }
}
```

This method should be moved to `List<Task>` because it works on the collection of `Task`. This smell can be called [Incomplete library class](https://refactoring.guru/smells/incomplete-library-class)
or [Primitive obsession](https://refactoring.guru/smells/primitive-obsession).

Now, we have a reason to introduce `Tasks` abstraction with the `format` method. Moving the `format` method into this new class is 
straightforward, however, we need to ensure that the change is small.

Look at the `format` method and notice the for-loop over `tasks` variable. With the new abstraction that we create, we should be able to use 
the same loop. To do this, our `Tasks` class should extend from `ArrayList<Task>`.

The `Tasks` class is defined as follows:

```java
public class Tasks extends ArrayList<Task> {
    void format(Writer writer) throws IOException {
        for (Task task : this) {
            writer.write(task.format());
        }
    }
}
```

And the `show` method:

```java
private void show() throws IOException {
    for (Map.Entry<String, Tasks> project : projects.entrySet()) {
        writer.write(project.getKey());
        writer.write("\n");
        Tasks tasks = project.getValue();
        tasks.format(writer);
    }
}
```

With this change, we had to replace the `List<Task>` with `Tasks`. To give an example,

```java
private final Map<String, List<Task>> projects = new LinkedHashMap<>();
```
became

```java
private final Map<String, Tasks> projects = new LinkedHashMap<>();
```

Run the tests and commit the change. Looking at the `format` method in the `Tasks` class, we can see that it both `formats` the tasks 
and writes the output to an instance of `java.io.Writer`. The `format` method should only be responsible for generating and returning 
a formatted `String`, while the act of writing should be handled separately.

This also aligns with the vocabulary of the `format` method that we introduced in [Recognize Code Smells](#recognize-code-smells) section.

The refactored code is shown below:

```java
public class Tasks extends ArrayList<Task> {
    String format() {
        return this.stream().map(Task::format).collect(Collectors.joining());
    }
}
```

The `show` method:

```java
private void show() throws IOException {
    for (Map.Entry<String, Tasks> project : projects.entrySet()) {
        writer.write(project.getKey());
        writer.write("\n");
        Tasks tasks = project.getValue();
        writer.write(tasks.format());
    }
}
```

Run the tests, ensure nothing breaks, add tests for the `format` method in `Tasks` and commit the change.