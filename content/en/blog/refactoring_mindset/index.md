---
author: "Sarthak Makhija"
title: "Refactoring Mindset"
date: 2025-01-11
description: "
"
tags: ["Refactoring", "Mindset", "Code Smells", "Clean Code"]
thumbnail: "/refactoring-mindset-title.jpg"
caption: "Photo by Magda Ehlers on Pexels"
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

The `Task` and `TaskList` are shown below:

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

#### Identify similar patterns

It is important to look for similar concepts in the code while refactoring. With the introduction of the `Tasks` abstraction
we can check if there are other places where this abstraction can be applied. The simplest way would be to look for the same pattern: for-loop over
a collection of `Task`. This leads to a **loose guideline**: a for-loop over a collection of type `T`, **may** become a static method which depends only
on the collection. This gives us an opportunity to [encapsulate the collection](https://refactoring.guru/encapsulate-collection).

Let's see if there is another method where `Tasks` can be used. Consider the `setDone` method, it contains the same pattern, for-loop
over a collection of `Task`.

```java
private void setDone(String idString, boolean done) {
    int id = Integer.parseInt(idString);
    for (Map.Entry<String, Tasks> project : projects.entrySet()) {
        for (com.codurance.training.tasks.Task task : project.getValue()) {
            if (task.getId() == id) {
                task.setDone(done);
                return;
            }
        }
    }
    out.printf("Could not find a task with an ID of %d.", id);
    out.println();
}
```

We can extract the inner for-loop in a separate method `toggleTaskWithId`.

```java
private void setDone(String idString, boolean done) {
    int id = Integer.parseInt(idString);
    for (Map.Entry<String, Tasks> project : projects.entrySet()) {
        Tasks tasks = project.getValue();
        if (toggleTaskWithId(done, tasks, id)) return;
    }
    out.printf("Could not find a task with an ID of %d.", id);
    out.println();
}

private static boolean toggleTaskWithId(boolean done, Tasks tasks, int id) {
    for (Task task : tasks) {
        if (task.getId() == id) {
            task.setDone(done);
            return true;
        }
    }
    return false;
}
```

The method `toggleTaskWithId` becomes static, and we can move it to the `Tasks` class. 

```java
public class Tasks extends ArrayList<Task> {
    boolean toggleTaskWithId(boolean done, int id) {
        for (Task task : this) {
            if (task.getId() == id) {
                task.setDone(done);
                return true;
            }
        }
        return false;
    }
    //not showing the format method
}
```

The method `toggleTaskWithId` method takes a `boolean` parameter. This might tempt us to refactor further by creating two separate 
methods: one to mark a task as done and another to mark it as undone. However, it’s important to stay focused on the current refactoring 
goal and avoid getting sidetracked.

Instead, this is a good opportunity to add a _TODO_ note, either directly in the code or in a separate file, as a reminder for a future
improvement. Run all the tests to ensure the changes didn’t break any functionality, add specific tests for the `toggleTaskWithId` method
and prepare the commit.

Let's take a look at the `show` method again.

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

We are seeing the same pattern again, a for-loop over a collection. We can extract the for-loop into a separate method.

```java
private void show() throws IOException {
    format(projects, writer);
}

private static void format(Map<String, Tasks> projects, Writer writer) throws IOException {
    for (Map.Entry<String, Tasks> project : projects.entrySet()) {
        writer.write(project.getKey());
        writer.write("\n");
        Tasks tasks = project.getValue();
        writer.write(tasks.format());
    }
}
```

The static method `format` works on a collection of type `Map<String, Tasks>` referred to as `projects`. Thus, the method can be moved 
to `Map` or we can encapsulate the `Map`. To proceed, we will now introduce the abstraction `Projects` and to keep the refactoring 
small, we will extend it from `LinkedHashMap<String, Tasks>`.

The `Projects` class is shown below:

```java
public class Projects extends LinkedHashMap<String, Tasks> {
    String format() {
        StringBuilder formatted = new StringBuilder();
        for (Map.Entry<String, Tasks> project : this.entrySet()) {
            formatted.append(project.getKey());
            formatted.append("\n");
            formatted.append(project.getValue().format());
        }
        return formatted.toString();
    }
}
```

The `show` method and the instantiation of `projects` variable look like the following:  

```java
private final Projects projects = new Projects();

private void show() throws IOException {
    this.writer.write(this.projects.format());
}
```

Run all the tests to ensure the changes didn’t break any functionality, add specific tests for the `format` method in `Projects`
and prepare the commit. It’s important to note that the `Projects` abstraction wasn’t introduced as an arbitrary design decision, 
it was driven by a clear need to encapsulate the collection. A similar refactoring can be applied to the `setDone` method because it has a 
for-loop over `projects.entrySet()`.

#### Learn to defer

It is crucial to not get carried away during refactoring. While it may be tempting to introduce a `Project` abstraction alongside 
`Projects`, doing so can lead to compile time errors. For instance, the updated `Projects` class might look like this:

```java
public class Projects extends LinkedHashMap<String, Project> {
    ...
}

class Project {
    final String name;
    final Tasks tasks;
    ...
}
```

The moment we do this, all the other methods start giving compile-time error. For example, consider the following method `addTask`.
The line `Tasks projectTasks = projects.get(projectName);` will no longer compile because `projects.get(..)` now returns a 
`Project` instead of `Tasks`: 

```java
private void addTask(String projectName, String description) {
    Tasks projectTasks = projects.get(projectName);
    if (projectTasks == null) {
        throw new IllegalArgumentException("Unknown project: " + projectName);
    }
    projectTasks.add(new com.codurance.training.tasks.Task(nextId(), description, false));
}
```

We can defer this refactoring, add a TODO, and revisit it later when the impact of changing `Projects` from being a `LinkedHashMap<String, Tasks>` to
`LinkedHashMap<String, Project>` is localized. This means, only the methods of the `Projects` class would be impacted. That would also be the
right time to evaluate whether `Projects` should extend `LinkedHashMap<>` or instead use composition. Here is my final version of the 
[Projects](https://github.com/SarthakMakhija/task-list-refactoring/blob/main/src/main/java/com/codurance/training/tasks/Projects.java) class.

#### Maintain a TODO list

It is crucial to maintain a TODO list while refactoring. Preparing a good TODO list has a lot of advantages: 

1. **Improved Focus**: Concentrating on individual tasks allows maintaining a steady pace and minimizing the feeling of being overwhelmed.
2. **Reduced Anxiety**: A well-defined TODO list minimizes anxiety by providing a clear path.
3. **Enhanced Visibility**: A detailed TODO list gives a clear view of all pending tasks, ensuring nothing gets overlooked.

I prefer adding refactoring TODOs directly in my code, where I document everything from ideas to potential distractions. 
These TODO items are removed as they are completed. An example is available in this [git commit](https://github.com/SarthakMakhija/task-list-refactoring/commit/ec553e57e8fcdc6adf87d75d41981efd773075d7).
At the end of the story, I make it a point to review and ensure no TODO is left unresolved.

#### Avoid coding in brain

#### Avoid bias  

#### Rubber duck

#### 