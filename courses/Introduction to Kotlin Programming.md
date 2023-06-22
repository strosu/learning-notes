# O'Reilly Introduction to Kotlin Programming

https://learning.oreilly.com/videos/introduction-to-kotlin/9781491964125/

Contents
========

 * [General](#general)
 * [Variables](#variables)
 * [Loops](#loops)
 * [Branching and comparison](#branching)
 * [Functions](#functions)
 * [Classes](#classes)

# General

- can target both the JVM, as well as JS and others
- no need to mark line endings with ;

# Variables

Two keywords for defining variables: 
- var, for mutable values
- val, for immutable values
    - val const for true compile-time constants, similar to C#

The type is usually infered by the compiler, so var x = 3 should be fine. 
When the compiler cannot infer, just append a suffix (literal constant), e.g. 10L is a Long type, 100F is Float with a value of 100

There is no implicit conversion, even when it would be safe to do so, e.g. an int to a long. Instead, helper functions are used, e.g. .toLong()

Primitives are capitalized, so Int, String, Boolean etc

**All variables are reference types**; the compiler figures out the performance details and uses primites whenever possible


# Loops


## Ranges

Ranges offer a nice solution for stuff like x >= 2 && x <=5, which can be rewritten as ``` x in 2..5 ``` (both ends of the range are considered as included)

Use a..b for ranges, e.g. 1..100
- for (a in 1..100) is the equivalent of a foreach
- for (b in 1..100 step 5) is the equivalent form of a for - start, end and increment

While loops are very standard:

var i = 100
while (i > 0) {
    i--
}

Break can be used to exit the nearest enclosing loop early. Alternatively, we can break to a specific parent, but need labels in this case.

Continue is used to move to the next iteration in the nearest enclosing loop. 

# Branching

Equality comparers:
- == for value equality
- === for reference equality (values point to the same object)
- !== for reference inequality

Composed conditional operators: same as C#

Conditional expressions (C# equivalent of var x = a > 2 ? 5 : 10)
- uses an explicit if-else (note the return is omited)
    
```
    var x = if(a > 2) {
        5
    }
    else {
        var y = 3
        if (y < 5) {
            smaller
        }
        else {
            larger
        }
    }
```

## When

The equivalent of switch in C# 
- uses -> instead of :
- no separator between conditions
- requires an else to be included (equivalent of default) when used for assignment
- can be used outside of an assignment, like a regular switch
- seems to allow mixed return types (e.g. string and int); this transforms the returned object into a Comparable and Seriaziable, while losing the intended type
    - doesn't seem like a good idea in the first place, perhaps might come in handy for nullable types
    - compiler figures out the actual type (e.g. String, String? etc)

```
val faction = when(otherVariable)
    "firstValue" -> "firstOutcome"
    "secondValue" -> "secondOutcome"
    "third" -> 1234
    else -> "none"
```

- multiple breanches might evaluate to true, but just the first one is picked. The code below results in "three", but swapping the first two conditions results in "two"

```
    when (val whenBreaker = 3) {
        3 -> print("three")
        in 2..4 -> print("two")
        5 -> print("five")
        in 6..10 -> println("Level $whenBreaker character")
    }
```

# Functions

### Single expression methods

Defined using the ***fun*** keyword.

Use = instead of =>. In most cases, the return type is inferred automatically, but sometimes the compiler needs a hand.

```
private fun doSmth(agument: String) : String = argument + "suffix"
```

## Return types

Specified at the end of the function signature, e.g. private fun doSmthh() : ***String***
**The equivalent of ***void*** is ***Unit***, which can be omitted**
For Unit methods, the return keyword can be skipped entirely. Using it results in an early exit of the method, just like C#.

Compared to void, Unit is compatible with generics. TODO - expand here once it's covered better

**Nothing** represents that nothing was retured. An example was of a method that simply threw an exception. Might be useful to check if a function execution returned Nothing?

## Arguments
- Can have defaults, specified via =
```
private fun forgeItem(itemName: String, material: String, quantity: Int = 1)
```
- no need to place optional arguments at the end of the list, however we need named arguments when calling and skipping the optional ones
```
private fun doStuff(first: String = "first", second: String)
```
- can be specified explicitly when invoked:
```
forgeItem(itemName = "stuff", "iron")
```

# Classes

- defined using the ***class*** keyword
- can contain properties, but the fields are not exposed directly

For instantiation, just call the constructor. Same as C#, without the **new** keyword.

No concept of static; instead of accessing methods without an actual object, a **companion** is defined. Seems to be a default instance of the class.
- supports having both functions and properties defined within, which result in the usual syntax for accessing them. 

```
class Book {
    companion object {
        var bookCount = 10;
        fun CreateBook() : Book = Book()
    }
}

fun main(args: Array<String>)
```

- the constructor signature and the property definition are merged, e.g. 
``` 
class customer(name: String, age: Int)
```

## Properties

- defining a property creates the backing field and getter/setter for it
- can be defined as a var or a as a val