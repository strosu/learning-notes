# Kotlin Programming: The Big Nerd Ranch Guide

Some personal notes, mostly comparing Kotlin to C# and focused on their differences

Contents
========

 * [General](#general)
 * [Variables](#variables)
 * [Branching and comparison](#branching)
 * [Functions](#functions)
 * [Error handling](#error-handling)
 * [Annonymous functions](#annonymous-functions)
# General

- can target both the JVM, as well as JS and others
- no need to mark line endings with ;

# Variables

Two keywords for defining variables: 
- var, for mutable values
- val, for values that do not change (readonly equivalent?)
    - val const for true compile-time constants, similar to C#

The type is usually infered by the compiler, so var x x= 3 should be fine

Primitives are capitalized, so Int, String, Boolean etc


**All variables are reference types**; the compiler figures out the performance details and uses primites whenever possible

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

## Ranges

Ranges offer a nice solution for stuff like x >= 2 && x <=5, which can be rewritten as ``` x in 2..5 ``` (both ends of the range are considered as included)

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

Use = instead of =>. In most cases, the return type is inferred automatically, but sometimes the compiler needs a hand.

```
private fun doSmth(agument: String) : String = argument + "suffix"
```

## Return types

Specified at the end of the function signature, e.g. private fun doSmthh() : ***String***
**The equivalent of ***void*** is ***Unit***, which can be omitted**
For Unit methods, the return keyword can be skipped entirely. Using it results in an early exit of the method, just like C#.

Compared to void, Unit is compatible with generics. TODO - expand here once it's covered better

## Arguments
- Can have defaults, specified via =
```
private fun forgeItem(itemName: String, material: String, quantity: Int = 1)
```
- no need to place optional arguments at the end of the list (although it is recomended). We need named arguments when calling and skipping the optional ones
```
private fun doStuff(first: String = "first", second: String)
```
- can be specified explicitly when invoked via = instead of :
```
forgeItem(itemName = "stuff", "iron")
```
- best practice is to either pass ALL argument names, or NONE of them

## Naming
- same as C# mostly
- can use ` (back-ticks) to surround the name of a function, in which case it's free for all in terms of names. The following are valid (and might come in handy for tests):
```
private fun `do some interesting stuff with user data, then delete it`()
fun doStuff() {
    `is`() // Invokes the Java is() method from Kotlin (is being a reserved word here)
}
```

# Strings

- are immutable, a new one is created every time we perform an operation on an existing string
- the == operator checks for ***string equality**, that is, both strings have the same content

- for interpolation, ${expression} is used, where anything goes inside the expression
- special characters need to be escaped, e.g. ", tabs, etc

## Raw strings
- can be used instead of interpolation in more complex scenarios
- nothing needs / can be escaped inside them
- enclosed with 3 quotation marks (""")
- fully preserves spacing inside
- to mark the start of a new line, use |
- use in conjunction with .trimMargin() to remove both the whitespace before and the | sign itself, resulting in more readable indentation

# Null safety
- enforced at compile time: all types are reference types and none can take a null by default
- vars need to be made explicitly nullable, i.e. **String?**

Smart casting:
- when using a nullable variable inside an if that checks it's not null, it automatically gets cast to the relevant non-nullable type

Safe call operator:
- instead of direct method invocation via ".", use "?.". At runtime, ths will return null and not evaluate the method if the object it's being ran against is null

Null coallecing operator:
- use ?: to specify an alternative value if the left hand side is null:
var x = getStuff() ?: "no stuff was returned"

Non-null assertion operator:
- use !! to force a bypass of the compiler safety; however, getting a null will throw 

# Error handling

- classical try/catch, nothing fancy
- instead of manually throwing, some keywords can help with that:
```
require(playerLevel > 0) {
    "Has to be greater than 0"
}
```

# Annonymous functions

Annonymouns functions are defined by enclosing in {} brackets, no need for () => like C#
- to invoke the method, we still need to call it via ()
```
        println({
            val exclamationMarksCount = 3
            message.uppercase() + "!".repeat(exclamationMarksCount)
        }())
```

- Type looks like () -> String (for a function that takes no arguments and returns a string)
    - it can be inferred usually, so no need for an explicit declaration

## Lambda functions
- uses -> instead of =>, e.g. x -> x in 'a'..'z'
- requires being surrounded by brackets, **outside** of the lambda
    - the body does not allow wrapping in brackets
- don't require a return, which is implcit
- don't seem to allow return, instead the last line is considered as the right value? (can be tricky, revisit)
- in arguments are passed as a list inside the paranthesis
```
        var withArguments: (String, Boolean) -> String = { input: String, _: Boolean ->
            input.uppercase()
        }
```

- lambdas can be passed as arguments to other functions, just like C#
- when a lambda is the last argument to a function, it can be placed outside of the paranthesis. This is especially useful when it's the only argument, in which case the () paranthesis can be skipped entirely

```
    narrate("Some new message here", narrationModifier)

    narrate("teeext") { message -> "$message and then some" }
```

## It identifier
- uses to identify the single argument that is passed into a lambda
```
    var withItArgument: (String) -> String = {
        it.lowercase(Locale.getDefault())
    }
```
- useful for very short lambdas, probably best avoided otherwise
```
"abcd".count({ it == 'a' })
```

## Multiple arguments
- the types can be passed either when defining the lambda, e.g. (String, Int) -> String

```
    var multipleArgs2: (String, Int) -> String = {
            message, counter ->
        message + counter
    }
```

- alternatively, they can be defined on the arguments themselves
```
    var multipleArgs = {
        message: String, counter: Int ->
        message + counter
    }
```