---
author: "Sarthak Makhija"
title: "Diving into Java Bytecode"
date: 2021-04-04
description: "Java code is compiled into an intermediate representation called \"bytecode\". It is this bytecode which gets executed by JVM and is later converted into machine specific instructions by JIT compiler. With this article, we attempt to dive into bytecode and understand the internals of various bytecode operations."
tags: ["Java", "JVM", "Bytecode"]
thumbnail: /diving-into-java-bytecode.png
caption: "Background by Kalle Kortelainen on Unsplash"
---

<blockquote class="wp-block-quote">
    <p>Java code is compiled into an intermediate representation called "bytecode". It is this bytecode which gets executed by JVM and is later converted into machine specific instructions by JIT compiler. With this article, we attempt to dive into bytecode and understand the internals of various bytecode operations.</p>
</blockquote>

Let's get an understanding of some terms before we start to dive in.

### Terminology

**Bytecode**

An intermediate representation of Java code which JVM understands.

<blockquote class="wp-block-quote">
    <p>This intermediate representation is called bytecode because each "opcode" is represented by 1 byte. This effectively means, a total of 256 opcodes are possible.</p> 
</blockquote>

These opcodes may take arguments and arguments can be up to 2 bytes long. This means a bytecode instruction which is a combination of an opcode and arguments could be as long as 3 bytes.

We will see various opcodes as we move on, but let's take a quick glimpse of an instruction which is an output from **javap** utility -
```java
9: iconst_0
```

- iconst_0 is an opcode which pushes a constant value 0 on top of the stack
- Every opcode is prefixed with a letter like ```i``` / ```d``` etc to represent the data type that opcode is dealing with
- Every bytecode instruction will start with an offset *(9: in the previous example)*. This comes handy when a "goto" opcode is used

**javap**

Standard Java class file disassembler distributed with JDK. It provides a human-readable format of class file.
```java
javap -p -v -c <path to the class file>

-p => display private methods
-v => be verbose
-c => disassemble the source code
```

### Quick overview of class file structure

Let's take a quick look at the structure of the class file. Don't worry if something is not clear at this stage, it should become clear as we proceed with examples.
Let's take a simple example to understand what constitutes our class file.

(*Please note: bytecode is trimmed for the entire article*).

```java
final public class SumOfN implements Serializable {
    private final int n;
    public SumOfN(int n) {
        this.n = n;
    }
    public int sum() {
        //code left out
    }
}
```

**bytecode**

```java
public final class SumOfN implements java.io.Serializable
   minor version: 0
   major version: 59
   flags: (0x0031) ACC_PUBLIC, ACC_FINAL, ACC_SUPER
   this_class: #8                          // org/sample/SumOfN
   super_class: #2                         // java/lang/Object
   interfaces: 1, fields: 1, methods: 2, attributes: 1
   Constant pool:
      #2 = Class              #4             // java/lang/Object
      #4 = Utf8               java/lang/Object
      #8 = Class              #10            // org/sample/SumOfN
      #10 = Utf8              org/sample/SumOfN
```

**Magic number (0xCAFEBABE)** is what every class file starts with. The first four bytes indicate that it is a class file and, the remaining four bytes
indicate the minor and major versions used to compile the source file.

**Major and minor version** indicate the version of JDK used to compile the source file. In the previous example minor version is 0 and major version is 59 (which is Java SE 15).

**Flags** indicate the modifiers that are applied to the class. In the previous example, we have ACC_PUBLIC indicating it is a public class,  ACC_FINAL indicating
it is a final class, ACC_SUPER exists for backward compatibility for the code compiled by Sun's older compilers for the Java programming language. (More on this [here](https://bugs.java.com/bugdatabase/view_bug.do?bug_id=6527033))

**Constant pool**
Is a part of class file which contains -
- string literals/constants
- doubles/float values
- names of classes
- interfaces
- fields

which are used in a class. Various opcodes like **invokevirtual** refer to constant pool entry to identify the virtual method to be invoked.
```java
1: invokevirtual #7  // Method run:()Ljava/lang/Object;
```
Here, **invokevirtual** takes an argument which refers to an entry in the constant pool and, the entry indicates the method to be called along with its parameter and return type.

**this_class** refers to an entry (#8) in the constant pool, which in turn refers to another entry (#10) in the pool that returns ```org/sample/SumOfN```.
Effectively, **this_class** holds the name of the current class.

**super_class** refers to an entry (#2) in the constant pool, which in turn refers to another entry (#4) in the pool that returns ```java.lang.Object```
Effectively, **super_class** holds the name of the super class.

**interfaces, fields, methods** respectively indicate the number of interfaces implemented by the class, number of fields that the class holds and the number of methods
that the class has.

**attributes** are used in the class file, field level information, method information, and code attribute structures. One example of an attribute is ```Exceptions```
which indicates which checked exceptions a method may throw. This attribute is attached to ```method_info``` structure.

This was a very quick overview of class file structure, for more details please check [this](https://docs.oracle.com/javase/specs/jvms/se16/html/jvms-4.html) link.

### Bytecode execution model

JVM operates using stack as its execution model. Stack is a collection of frames, each of which is allocated when a method is invoked.

A stack frame consists of -

**Operand Stack**
Most of the opcodes operate by pushing-in or popping-out value to or from the operand stack. Eg; **iconst_0** pushes 0 on top of the stack.

**LocalVariableTable (an array of local variables)**
In order to allow a variable to be assigned a value, a local variable table is used. ```LocalVariableTable``` is a simple data structure which contains the name of the variable,
its data type, its slot along with some other fields.

LocalVariableTable contains -
- variables that are local to a method
- method parameters
- ```this```, if the method is not static. ```this``` is allocated slot 0 in LocalVariableTable

Eg; **istore_1** is an opcode which stores an integer value into LocalVariableTable at slot 1 by picking value from top of the stack.

### Introducing bytecode opcodes
Let's take a simple example which adds 2 integers, to understand opcodes and their execution.

**AdditionExample**

```java
public class AdditionExample {
    public int execute() {
        int addend = 10;
        int augend = 20;
        return addend + augend;
    }
}
```

**bytecode (AdditionExample)**

```java
public class AdditionExample {
public AdditionExample();
Code:
0: aload_0
1: invokespecial #1   // Method java/lang/Object."<init>":()V
4: return

    public int execute();
        Code:
          stack=2, locals=3, args_size=1
            0: bipush        10
            2: istore_1
            3: bipush        20
            5: istore_2
            6: iload_1
            7: iload_2
            8: iadd
            9: ireturn

        LocalVariableTable:
        Start  Length  Slot  Name     Signature
            0      10     0  this     Lorg/sample/AdditionExample;
            3       7     1  addend   I
            6       4     2  augend   I
}
```

The bytecode of ```AdditionExample()``` should become clear as we move on but first let's understand the bytecode of ```execute``` method -
1. The Java compiler has indicated the depth of stack needed during the execution of this method. ```stack=2``` means at any point during this method execution
   we will have a maximum of 2 entries on the stack. ```locals=3``` indicate that there are 3 local variables which will need to go in LocalVariableTable. One variable is
   ```addend```, other is ```augend``` and the last is ```this```. ```args_size=1``` indicates one object needs to be initialized before the method call, which
   again is ```this```
2. **bipush** is an opcode which pushes a byte sized integer on the stack. It takes an argument which is 10 in our case
3. **istore_1** takes the value from top of the stack, which is 10 and assigns it into LocalVariableTable at slot 1. This opcode removes the value from stack top
4. **bipush** now pushes 20 to the top of the stack
5. **istore_2** takes the value from top of the stack, which is 20 and assigns it into LocalVariableTable at slot 2
6. At this stage, values 10 and 20 have been assigned to addend and augend in LocalVariableTable, and our stack is empty. This means these 2 values need to be brought into stack before an addition can be performed
7. **iload_1** copies the value from slot 1 of LocalVariableTable to the stack
8. **iload_2** copies the value from slot 2 of LocalVariableTable to the stack
9. Stack now contains 10 and 20. **iadd** pops 2 integer values from top 2 positions of the stack and sums them up. It stores the result back in the stack top
10. **ireturn** takes the value from stack top and returns an integer

Following diagram represents the overall execution -
<div class="wp-block-image is-style-default">
    <img src="/addition-example.png" />
</div>
<p></p>

Few things to note-
- All the opcodes are prefixed with an ```i```, indicating that we are dealing with an integer data type
- Slot 0 of LocalVariableTable is occupied by ```this``` of AdditionExample
- All the entries in LocalVariableTable are statically typed
- Bytecode is statically typed in a sense that all the opcodes which work with specific data type

Quick summary of opcodes that we have seen so far -

| Opcode  | Purpose |
| ------------- | ------------- |
| istore_slot  | Takes an integer value from top of the stack and assigns it into LocalVariableTable at defined slot  |
| iload_slot  | Copies the value from defined slot of LocalVariableTable to the stack  |
| bipush  | Pushes a byte sized integer on the stack |

Let's take another example, which is a slight modification of the first one. The idea is to invoke a method from another method.

**MethodInvocation**

```java
public class AdditionExample {
    public int execute() {
        return add();
    }
    private int add() {
        int addend = 10;
        int augend = 20;
        return addend + augend;
    }
}
```

**bytecode (MethodInvocation)**

```java
public class AdditionExample {
Constant pool:
#7 = Methodref          #8.#9          // org/sample/AdditionExample.add:()I
#8 = Class              #10            // org/sample/AdditionExample
#9 = NameAndType        #11:#12        // add:()I
#10 = Utf8              org/sample/AdditionExample
#11 = Utf8              add
#12 = Utf8              ()I

    public AdditionExample();
        Code:
            0: aload_0
            1: invokespecial #1    // Method java/lang/Object."<init>":()V
            4: return
        
    public int execute();
        Code:
            0: aload_0
            1: invokevirtual #7   // Method add:()I
            4: ireturn
    
    private int add();
        Code:
            0: bipush        10
            2: istore_1
            3: bipush        20
            5: istore_2
            6: iload_1
            7: iload_2
            8: iadd
            9: ireturn
}
```

Bytecode in ```add``` method should look very familiar 😁. Let's look at the bytecode for ```execute``` method which only invokes ```add``` method-
1. **aload_0** copies the value from slot 0 of LocalVariableTable to the stack. Slot 0 of LocalVariableTable contains ```this```, which means stack top now contains ```this```
2. Now is the time to invoke the private method ```add``` of the same class. **invokevirtual** is used for invoking a virtual method and, it takes a parameter which is a reference to an entry in the constant pool. Let's see how does this entry get used -
    - Entries in constant pool are composable, which means an entry could be created by referring to other entries
    - #7 is a method reference entry which refers to #8 and #9
    - #8 refers to an entry #10 which specifies the name of the class ```org/sample/AdditionExample```
    - #9 refers to entries #11 and #12 which specify the method name ```add``` along with its signature ```()I``` (*no parameters, integer return type*) respectively
    - #7 in short, provides a complete signature of the ```add``` method including its class name ```org/sample/AdditionExample.add:()I```
3. Our stack contains ```this``` which will be used for invoking ```add``` method
4. **invokevirtual** pops the entry from stack top which is ```this```, invokes ```add``` method and stores the result in stack top
5. **ireturn** takes the value from stack top and returns an integer

*javap by default does not return the output for private methods, use -p flag to see the output for private methods.*

One of the questions that is worth answering is "how does **invokevirtual** know about the number entries to be popped out?". In order to answer this, we will modify our
previous example slightly and see the behavior of **invokevirtual**.

**MethodInvocation with parameters**

```java
public class AdditionExample {
    public int execute() {
        return add(10, 20);
    }

    private int add(int addend, int augend) {
        return addend + augend;
    }
}
```

**bytecode (MethodInvocation with parameters)**

```java
public class AdditionExample {
public AdditionExample();
Code:
....

    public int execute();
        Code:
            0: aload_0
            1: bipush        10
            3: bipush        20
            5: invokevirtual #7       // Method add:(II)I
            8: ireturn
    
    private int add(int, int);
        Code:
            ...
}
```

Let's look at the bytecode for ```execute``` method again -
1. ```this``` is pushed on the stack, followed by push of values 10 and 20
2. Stack contains ```this```, ```10``` and ```20```
3. There is a change in signature of the method which will be invoked by **invokevirtual**. ```add``` now takes 2 integer parameters and returns an integer. Method signature is denoted by ```add:(II)I```
4. **invokevirtual** now needs to pop 3 entries from the stack, 2 integers which were pushed using **bipush** opcode and a reference to ```this``` which was pushed using **aload_0**
5. Once it pops the entries, ```add``` method is invoked and, the result is stored in stack top
6. **ireturn** takes the value from stack top and returns an integer

Effectively, **invokevirtual** knows the number of entries to be popped based on the signature of the method to be invoked. As seen in previous example, in order to invoke a method
which takes 2 parameters, we need to pop 2 values from the stack along with an instance of the current class.

Quick summary of opcodes that we have seen so far -

| Opcode  | Purpose |
| ------------- | ------------- |
| aload_slot  | Copies the address value from a defined slot of LocalVariableTable to the stack, ```a``` stands for address  |
| invokevirtual  | Invokes virtual method, pops the entries from stack based on the signature of the method to be invoked  |

### Opcodes for object creation
Let's take an example to understand the bytecode that gets generated during object creation.

```java
public class Book {
    public Book(String name, Date publishingDate) {
        ///
    }
    public Date publishingDate() {
        return new Date();
    }
}
```
This example uses ```java.util.Date```, (don't ask why) and returns a ```new Date``` as a part of ```publishingDate``` method (again, don't ask why 😁).

**bytecode (Object creation)**

```java
public class Book {
public java.util.Date publishingDate();
   Code:
   0: new           #7     // class java/util/Date
   3: dup
   4: invokespecial #9     // Method java/util/Date."<init>":()V
   7: areturn
}
```

This is a new territory that we are going into. Let's understand the bytecode -
1. **new** allocates the required memory for the object but does not call the constructor. It refers to the constant pool and identifies the object which is ```java/util/Date``` here, and allocates the required memory
2. Our stack now contains the object reference created by **new**
3. Before we understand **dup**, let' understand the need for it -
    - Let's assume there is no **dup**
    - Our stack contains an object reference which means it is referring to some memory allocated by **new** opcode
    - So far our ```date``` object has not been initialized. We need another opcode (invokespecial) for initializing it
    - **invokespecial** is used for invoking special methods like constructors. **invokespecial** refers to an entry in the constant pool (#9), resolves it
      to the init method of ```java.util.Date``` class.
    - **invokespecial** will pop the entry from stack top and invoke the ```init``` method of ```java.util.Date```. This means our date object is fully initialized now
    - But, now our stack does not contain any reference to the newly created object because it was popped by **invokespecial** to invoke a method which does not return anything
4. So, we need **dup** to duplicate the entry on stack top
5. **invokespecial** pops the entry from stack top, invokes the ```init``` method of ```java.util.Date``` class to initialize the object
6. We now have the stack containing an object reference which refers to the fully created ```java.util.Date``` instance
7. **areturn** takes the value from stack top and returns ```java.util.Date``` address

Following diagram represents the overall execution -
<div class="wp-block-image is-style-default">
    <img src="/new-dup.png" />
</div>
<p></p>

Quick summary of opcodes that we have seen so far -

| Opcode  | Purpose |
| ------------- | ------------- |
| new  | Allocates the required memory for the object does not call the object constructor |
| dup  | Duplicates the entry present on stack top |
| invokespecial  | Invokes special methods like constructors  |

### Combining things together
Time to take one last example and validate our learning.

```java
public class SumOfN {
    private final int n;
    public SumOfN(int n) {
        this.n = n;
    }
    public int sum() {
      int sum = 0;
      for (int number = 1; number <= n; number++) {
        sum = sum + number;
      }
      return sum;
    }
}
```

**bytecode (SumOfN) constructor**

```java
public class SumOfN {
    private final int n;

    public SumOfN(int);
        Code:
            0: aload_0
            1: invokespecial #1     // Method java/lang/Object."<init>":()V
            4: aload_0
            5: iload_1
            6: putfield      #7     // Field n:I
            9: return
        
        LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      10     0  this   Lorg/sample/SumOfN;
            0      10     1     n   I
}
```

Let's begin with ```SumOfN(int)``` constructor and understand the bytecode. Instead of going through the code first, let's try and, figure out what might the bytecode look like by
understanding what needs to be done.

<table style="width:100%">
  <tr>
    <th>What needs to be done</th>
    <th>How can it be done</th>
  </tr>
  <tr>
    <td rowspan="2">We should be able to invoke the constructor of <code class="language-plaintext highlighter-rouge">java.lang.Object</code></td>
    <td>load <code class="language-plaintext highlighter-rouge">this</code> reference on the stack, which is what <b>aload_0</b> does</td>
  </tr>
  <tr>
    <td>invoke <code class="language-plaintext highlighter-rouge">init</code> method of <code class="language-plaintext highlighter-rouge">java.lang.Object</code> which is what <b>invokespecial</b> does. It pops <code class="language-plaintext highlighter-rouge">this</code> reference from stack top</td>
  </tr>

  <tr>
    <td rowspan="3">We should be able to store the value of <code class="language-plaintext highlighter-rouge">n</code> in class field</td>
    <td>load <code class="language-plaintext highlighter-rouge">this</code> reference on the stack, which is what <b>aload_0</b> does</td>
  </tr>
  <tr>
    <td>load the value of <code class="language-plaintext highlighter-rouge">n</code> on the stack from LocalVariableTable, which is what <b>iload_1</b> does. <code class="language-plaintext highlighter-rouge">n</code> has slot 1 in LocalVariableTable</td>
  </tr>
  <tr>
    <td>put the value of <code class="language-plaintext highlighter-rouge">n</code> in class field, which is what <b>putfield</b> does. It pops the 2 entries from stack top, one is <code class="language-plaintext highlighter-rouge">this</code> and other is the value of <code class="language-plaintext highlighter-rouge">n</code> and sets the class field</td>
  </tr>
</table>

All these opcodes make up our constructor. Let's now jump to the ```sum``` method.

**bytecode (SumOfN) sum method**

```java
public int sum();
   Code:
      0: iconst_0
      1: istore_1
      2: iconst_1
      3: istore_2
      4: iload_2
      5: aload_0
      6: getfield      #7     // Field n:I
      9: if_icmpgt     22
      12: iload_1
      13: iload_2
      14: iadd
      15: istore_1
      16: iinc          2, 1
      19: goto          4
      22: iload_1
      23: ireturn

    LocalVariableTable:
    Start  Length  Slot  Name   Signature
        4      18     2  number   I
        0      24     0  this     Lorg/sample/SumOfN;
        2      22     1  sum      I
```

<table style="width:100%">
  <tr>
    <th width="30%">What needs to be done</th>
    <th width="30%">Code snippet</th>
    <th width="40%">How can it be done</th>
  </tr>
  <tr>
    <td>Initialize <code class="language-plaintext highlighter-rouge">sum</code> with a value 0</td>
    <td>int sum = 0</td>
    <td><b>iconst_0</b> and <b>istore_1</b> should be able to put 0 on the stack and assign it to local variable sum. <code class="language-plaintext highlighter-rouge">sum</code> variable has slot 1 in LocalVariableTable</td>
  </tr>
  <tr>
    <td>Initialize <code class="language-plaintext highlighter-rouge">number</code> with a value 1</td>
    <td>int number = 1</td>
    <td><b>iconst_1</b> and <b>istore_2</b> should be able to put 1 on the stack and assign it to local variable number. <code class="language-plaintext highlighter-rouge">number</code> variable has slot 2 in LocalVariableTable</td>
  </tr>
  <tr>
    <td>We should be able to compare the value of <code class="language-plaintext highlighter-rouge">number</code> and the value of the class field <code class="language-plaintext highlighter-rouge">n</code>. In order for this to happen, we need to load the 
   value of <code class="language-plaintext highlighter-rouge">number</code> and <code class="language-plaintext highlighter-rouge">this</code> reference on the stack. We need <code class="language-plaintext highlighter-rouge">this</code> reference to be able to get the value of instance variable <code class="language-plaintext highlighter-rouge">n</code></td>
    <td>number <= this.n</td>
    <td><b>iload_2</b> and <b>aload_0</b> should be able to copy the value of <code class="language-plaintext highlighter-rouge">number</code> variable from slot 2 and <code class="language-plaintext highlighter-rouge">this</code> reference from slot 0 on the stack</td>
  </tr>
  <tr>
    <td>We should be able get the value of class field <code class="language-plaintext highlighter-rouge">n</code></td>
    <td>this.n</td>
    <td><b>getfield</b> should be able to help here. It pops the class instance to get the field value. The field value goes on the stack. Now, our stack contains value of number and n</td>
  </tr> 
  <tr>
    <td>Perform the required comparison. If the condition indicates exit from the loop, return the value present on the stack top</td>
    <td>number <= this.n</td>
    <td><b>if_icmpgt</b> does the integer comparison. It pops the top 2 integer values from the stack and does the comparison (number > n). If condition returns true, it takes an argument which is the instruction offset to jump to</td>
  </tr> 
  <tr>
    <td>If the condition indicates loop continuation, load the value of <code class="language-plaintext highlighter-rouge">sum</code> and <code class="language-plaintext highlighter-rouge">number</code> variable to be able to perform addition</td>
    <td>sum + number</td>
    <td><b>iload_1</b> and <b>iload_2</b> should do it. Now our stack has 2 values which are ready for addition</td>
  </tr> 
  <tr>
    <td>Perform addition</td>
    <td>sum + number</td>
    <td><b>iadd</b> does the integer addition. Pops the top 2 values from the stack and puts the result back in the stack</td>
  </tr>
  <tr>
    <td>Assign the result of addition to the variable <code class="language-plaintext highlighter-rouge">sum</code></td>
    <td>sum = sum + number</td>
    <td><b>istore_1</b> would do the job. It takes the value from stack top and assign the value in variable sum</td>
  </tr>
  <tr>
    <td>Increment the value of <code class="language-plaintext highlighter-rouge">number</code></td>
    <td>number = number + 1</td>
    <td><b>iinc</b> does integer increment and takes 2 parameters. First one is the LocalVariableTable slot and other one is the increment. It is one of the opcodes that does not work 
    with stack. It increments the value at a specific slot in LocalVariableTable</td>
  </tr>
  <tr>
    <td>Repeat steps</td>
    <td>NA</td>
    <td><b>goto</b> is the opcode which transfers the control to a specific instruction set</td>
  </tr>
</table>

Following diagram represents the overall execution of ```sum``` method -
<div class="wp-block-image is-style-default">
    <img src = "/combining-things-together.png" />
</div>
<p></p>

### Summary
Let's conclude with some key takeaways -

- javap provides a human-readable format of class file
- Each opcode in a bytecode is represented by 1 byte
- Each opcode is prefixed with a letter indicating the data type the opcode will work with
- Most of the opcodes work with the stack which means before they operate, values need to be brought on the stack
- Opcode like **iinc** works with LocalVariableTable instead of working with values on the stack
- Opcodes like **invokevirtual**, **invokespecial** refer to an entry in the constant pool to resolve the method that needs to be invoked
- Some opcodes also have shortcuts. eg; **iconst_0** pushes 0 on the stack without taking any argument. It could have been designed to take an argument
  but that would have meant the total instruction size will be greater than 1 byte (1 byte for the opcode and another byte for the argument). In order to avoid
  increasing the size of the instruction, it is designed in a shortcut form

Hope it was meaningful. Appreciate the feedback.

### References
+ [Advanced Java Bytecode Tutorial](https://www.jrebel.com/blog/java-bytecode-tutorial)
+ [Java Bytecode: Using Objects and Calling Methods](https://www.jrebel.com/blog/using-objects-and-calling-methods-in-java-bytecode)
+ [Java Bytecode Crash Course](https://www.youtube.com/watch?v=e2zmmkc5xI0)
+ [Optimizing Java - practical techniques for improving JVM application performance](https://www.amazon.in/Optimizing-Java-Techniques-Application-Performance/dp/9352137132)