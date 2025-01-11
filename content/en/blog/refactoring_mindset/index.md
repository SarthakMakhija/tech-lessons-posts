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

