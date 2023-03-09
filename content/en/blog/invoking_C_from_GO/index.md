---
author: "Sarthak Makhija"
title: "Invoking C Code from Golang"
date: 2021-12-21
description: "Let's explore Golang's C package to invoke C code from Golang by building a linked list in C which would offer put, get, getAllValues, length and close features."
tags: ["Golang", "C", "CGO"]
thumbnail: /invoking-C-code-from-golang.png
caption: "Background by Giorgio Trovato on Unsplash"
---

The article attempts to explore Golang's "C" package which allows invoking C code from Golang. Before we get into the idea
of invoking C code from Golang, let's see a use-case where this might be needed.

### Interfacing with an existing C library
Let's consider that we wish to develop a storage engine for pmem (persistent memory) in Golang. In order to develop this, we might want to use
[pmdk - persistent memory development kit](https://github.com/pmem/pmdk) which is written in C. This effectively means we want a way to bridge Golang and C code; invoke C code
from Golang.

### "C" package in Golang
Go provides a package called "C" to interface with C code. Some features provided by this package include -
- standard C numeric types
- access to structs defined in C
- access to function like `malloc` and `free`

### C Code
Let's start by creating a linked list in C which will be later invoked from Golang code. Let's start with `linkedlist.h` file which defines a struct called `Node`, and a set of operations supported by linked list.

```C
struct Node {
    int key;
    int value;
    struct Node *next;
};

void put(int key, int value);
int get(int key);
```

Our `Node` struct has a key, and a value which are of type `integer` and a pointer to the next node. It also supports 2 behaviors `put` and `get`.

Let's add `linkedlist.c` and add `put` to begin with.

```C
#include "linkedlist.h"
#include <stdlib.h>

struct Node *head = NULL;
struct Node *current = NULL;

void put(int key, int value) {
    if (head == NULL) {
        head = (struct Node*)malloc(sizeof(struct Node));
        head -> key = key;
        head -> value = value;
        current = head;
    }
    else {
        struct Node *next = (struct Node*)malloc(sizeof(struct Node));
        next -> key = key;
        next -> value = value;
        current -> next = next;
        current = next;
    }
}
```

`put` does the following -
- If head is NULL
    - Creates a `new node`
    - Points `head` to the `new node`
    - Assigns key and value to the `head`
    - Points `current` to the `head`
- Else
    - Creates a `new node`
    - Assigns key and value to the `new node`
    - Assigns `next` of the `current` node to the `new node`
    - Points `current` to the `new node`

Let's add `get` which will return a value by key.

```C
int get(int key) {
    struct Node* node = head;
    while (node->key != key && node != NULL) {
        node = node -> next;
    }
    if (node == NULL) {
        return -1;
    }
    return node -> value;
}
```

`get` does the following -
- Makes a temporary variable `node` point to `head`
- Iterates while `node` is not NULL and `key of the node` is not equal to the incoming `key`
- If `node` is NULL, returns -1
- Else returns the value pointed by `node`

### Time to invoke C code from Golang

Let's create a file `linkedlist.go` which will provide `Put` and `GetBy` behaviors. These functions will internally invoke C functions.

```golang
package linkedlist

// #cgo CFLAGS: -g -Wall
// #include "linkedlist.h"
import "C"
```

This is how the beginning of our go file looks like -
- Go package is `linkedlist`
- Adding the line `#cgo CFLAGS: -g -Wall` compiles the C code with gcc options: (-g) which is used to enable debug symbols and (-Wall) which is used to enable all warnings
- `linkedlist.h` is included for our linked list related functions
- The import "C" allows us to integrate with C code
- The comments above `import C` represent the actual C code that will be consumed by the rest of our golang code

Let's add `Put` in Golang.

```golang
func Put(key, value int) {
    C.put(C.int(key), C.int(value))
}
```

Golang `Put` does the following -
- Creates a C int by using `C.int` on key and value
- Invokes `put` by using `C.put`, passing C.int

Let's add `GetBy`.

```golang
func GetBy(key int) int {
    return int(C.get(C.int(key)))
}
```

Golang `GetBy` does the following -
- Creates a C int by using `C.int` on key
- Invokes `get` by using `C.get`
- Convert the return value received from `C.get(C.int(key))` to `Golang's int`

*Putting it all together* -

```golang
package linkedlist

// #cgo CFLAGS: -g -Wall
// #include "linkedlist.h"
import "C"

func Put(key, value int) {
    C.put(C.int(key), C.int(value))
}

func GetBy(key int) int {
    return int(C.get(C.int(key)))
}
```

Let's add a Golang test to see if this integration works or not.

```golang
package linkedlist_test

import (
"reflect"
"testing"
)
import "linkedlist"

func TestPutsASingleKeyValue(t *testing.T) {
    linkedlist.Put(10, 100)

    value := linkedlist.GetBy(10)
    if value != 100 {
        t.Fatalf("Expected %v, received %v", 100, value)
    }
}
```

Let's Run the test using `go test`, and it should work.

### Freeing the linked list

We would like to add another test which adds multiple key value pairs and gets a value by key. Before we do that, we would like to start with a fresh linked list
for each test. In order to do that let's add a behavior `close` in C linked list which frees all the nodes in the linked list.

```C
void close() {
    struct Node* node;
    while (head != NULL) {
        node = head;
        head = head->next;
        free(node);
    }
}
```

`close` does the following -
- Makes a temporary variable `node`
- Iterates while `head` is not NULL
- Makes `node` point to head and moves `head` ahead
- Frees the memory pointed to by `node`

Let's invoke `C's close` from Golang

```golang
func Close() {
    C.close()
}
```

Let's edit the existing test case and add a new one.

```golang
func TestPutsASingleKeyValue(t *testing.T) {
    defer linkedlist.Close()
    linkedlist.Put(10, 100)

    value := linkedlist.GetBy(10)
    if value != 100 {
        t.Fatalf("Expected %v, received %v", 100, value)
    }
}

func TestPutsMultipleKeyValues(t *testing.T) {
    defer linkedlist.Close()
    linkedlist.Put(10, 100)
    linkedlist.Put(20, 200)
    linkedlist.Put(30, 300)

    value := linkedlist.GetBy(20)
    if value != 200 {
        t.Fatalf("Expected %v, received %v", 200, value)
    }
}
```

As a part of these tests, we are now closing the linked list by invoking `defer linkedlist.Close()` which will internally free all the nodes.

### Let's get all the values
We would like to return all the values from linked list. As a first step, we would like to return a pointer to an integer from C code to Golang.
Let's code `getAllValues` in C.

```C
int* getAllValues() {
    int len = length();
    int* values = malloc(len * sizeof(int));
    struct Node* node = head;
    int index = 0;

     while (node != NULL) {
         values[index] = node -> value;
         node = node -> next;
         index = index + 1;
     }
     return values;
}
```

`getAllValues` does the following -
- Invokes `length` to get the total number of nodes in the linked list
- Allocates memory to hold `len` integer values in a variable called `values`
- Creates a pointer called `node` to point to `head`
- Iterates while `node` is not NULL
- Collects the value pointed to by `node` in `values`
- Returns `values` which is a pointer to an integer

Let's also add `length` function -

```C
int length() {
    int count = 0;
    struct Node* node = head;
    while (node != NULL) {
        node = node -> next;
        count = count + 1;
    }
    return count;
}
```
`length` does the following -
- Creates a pointer called `node` to point to `head`
- Iterates while `node` is not NULL
- Increments count for each node
- Returns count

Time to invoke `getAllValues` from Golang.

```golang
func GetsAllValues() []int {
    length := int(C.length())
    intPointer := C.getAllValues()
    defer C.free(unsafe.Pointer(intPointer))

	slice := (*[1 << 28]C.int)(unsafe.Pointer(intPointer))[:length:length]

	var values []int
	for index := 0; index < length; index++ {
		values = append(values, int(slice[index]))
	}
	return values
}
```

`GetAllValues` does the following -
- Invokes `C.length` to get the total number of linked list nodes
- Invokes `C.getAllValues()` to get a pointer to an integer
- We would like to free the memory held by the pointer returned from `C.getAllValues()`. In order to do that, we need to invoke `C.free` and
  we use `unsafe.Pointer` which allows creating a pointer to any arbitrary type
- In order to be able to invoke `C.free` or `C.malloc`, we need to import `stdlib.h` in Golang code
- So far we have a pointer or let's say a C pointer. We need a way to convert that to Go slice.

The expression `slice := (*[1 << 28]C.int)(unsafe.Pointer(intPointer))[:length:length]` is made up of 3 parts -

```golang
1. unsafePointer := unsafe.Pointer(intPointer) creates an unsafe pointer

2. arrayPtr := (*[1 << 28]C.int)(unsafePointer).`*[1 << 28]C.YourType` doesn't do anything itself, it is a type.
   Specifically, it is a pointer to an array of size 1 << 28, of C.YourType values.
   Statement 2) converts unsafePointer to a pointer of the type *[1 << 28]C.int

3. slice := arrayPtr[:length:length], slices the array into a Go slice
```

>[what-does-1-30c-yourtype-do-exactly-in-cgo](https://stackoverflow.com/questions/48756732/what-does-1-30c-yourtype-do-exactly-in-cgo)

- We now have a Golang slice of type `C.int` but we need to return a Golang slice of `Go int`. So we iterate through the `slice`, convert `C.int` to `Golang int`,
  collect `values` and return it

Let's quickly add a golang test for `GetAllValues`.

```golang
func TestGetAllValues(t *testing.T) {
    defer linkedlist.Close()
    linkedlist.Put(10, 100)
    linkedlist.Put(20, 200)
    linkedlist.Put(30, 300)

    values := linkedlist.GetAllValues()
    expected := []int{100, 200, 300}
    
    if !reflect.DeepEqual(expected, values) {
        t.Fatalf("Expected %v, received %v", expected, values)
    }
}
```

That is it, run the test and get all the values from linked list.

### Code
The code for this article is available [here](https://github.com/SarthakMakhija/CGO-learning)

### References
- [Calling C code from go](https://karthikkaranth.me/blog/calling-c-code-from-go/)
- [Golang slices and C pointers](https://stackoverflow.com/questions/64852226/how-to-iterate-through-a-c-array)
- [What does 1-30c-yourtype do exactly in cgo](https://stackoverflow.com/questions/48756732/what-does-1-30c-yourtype-do-exactly-in-cgo)
- [ld: symbol(s) not found for architecture x86_64](https://github.com/golang/go/issues/31409)