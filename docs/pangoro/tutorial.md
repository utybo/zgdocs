# Tutorial/guide

Welcome to Pangoro! This page will guide you through your first steps using the Pangoro parser.

?> For instructions on how to get Pangoro, see [Getting Started](start.md).

## Parser 101

A parser is a tool that turns a list of tokens into an [abstract syntax tree (AST)](https://en.wikipedia.org/wiki/Abstract_syntax_tree).

Pangoro allows you to do just that: based on Lixy tokens, Pangoro will turn them
into an abstract syntax tree with an easy to use syntax.

## Deciding on our AST's structure

Creating a parser is a little bit more involved than creating a lexer in the
statically-typed world, specifically because we need to create the elements of
the AST first.

The actual structure of your AST is entirely up to you: Pangoro's only
constraint is that the nodes must have implement the `PangoroNode` marker
interface and must have a `PangoroNodeDeclaration<T>` somewhere, where `T` is
that node. The recommended method is to have that declaration be the companion
object of the node class, but you can also do whatever you prefer.

Your AST is a (typed) tree. All of the classes of Kotlin could be useful here,
but sealed class can be especially useful. For example, an expression could be a
string, a number, a function invocation or an operation on multiple other
expressions: using a sealed class, we could have a clean class hierarchy for all
of these cases.

## Declaring a node

Every node of the AST must:

* Implement `PangoroNode`
* Have a `PangoroNodeDeclaration` somewhere

The recommended way of creating a simple node is:

```kotlin
class MyNode(...) : PangoroNode {
    companion object : PangoroNodeDeclaration<MyNode> by reflective()
}
```

This way, reflection will be used to populate your constructor with arguments
stored during the parsing process (we'll see how to do that later on). You can
also build your nodes by manually implementing `PangoroNodeDeclaration`.

```kotlin
class MyNode(...) : PangoroNode {
    companion object : PangoroNodeDeclaration<MyNode> {
        override fun make(args: PangoroTypeDescription): MyNode =
            MyNode(...)
    }
}
```

You can retrieve the stored arguments using `args["name"]`.

!> Reflection is not supported on Kotlin/JS. Use the second variant if you are
targeting Kotlin/JS or MPP.

?> `reflective()` supports classes with multiple constructors, although this is
not really idiomatic in Kotlin

## Describing a node

Now that we *declared* our node and how to build it, it's time to tell Pangoro
how to parse it!

This is actually very simple, provided that you are using the `companion object`
as described above.

```kotlin
val parser = pangoro {
    MyNode {
        // ...
    }
}
```

And... That's it! You can now put your expectations in the `MyNode` block.

## Expectations

Pangoro is based heavily on the concept of "expectations".

Each node "expects" to match a specific structure. For example, an "assignment" might look like

```
myVar = "Hello!";
myOtherVar = 31;
```

After a lexer, we get the following tokens:

* `myVar` (`element`), `=` (`opAssign`), `"Hello!"` (`stringValue`), `;`
  (`semicolon`)
* `myOtherVar` (`element`), `=` (`opAssign`), `31` (`intValue`), `;`
  (`semicolon`)

Roughly speaking, our "Assignment" node will match against something like this:

```
                     /-> stringValue \
element -> opAssign -                 -> semicolon
                     \-> intValue    /
```

We need to store some useful information too, specifically the "element" and the
assignment's value (a string or an int). The other tokens are only syntactic
stuff which we don't really need in our abstract syntax tree. Let's store our
element name in "element" and our value in "value".

We can then set up some expectations to express that structure in code:

```kotlin
Assignment {
    expect(element) storeIn "element"
    expect(opAssign)
    either {
        expect(stringValue) storeIn "value"
    } or {
        expect(intValue) storeIn "value"
    }
    expect(semicolon)
}
```

This is a bit verbose. We could also use a shorter variant:

```kotlin
Assignment {
    +element % "element"
    +opAssign
    either {
        +stringValue % "value"
    } or {
        +intValue % "value"
    }
    +semicolon
}
```

?> `+x` is a shortcut for `expect(x)` and `%` is a shortcut for `storeIn`. Most
of the examples in this guide will use the more verbose variants for clarity.

We now have a description of our node based on expectations! Each line in there
is an expectation. Expectations come in many shapes, and some can even store
their results.


Notice the `storeIn`/`%`, they describe which values should be stored. These
values will be served directly to the `PangoroNodeDeclaration` in the
arguments. Here, our expectations simply expect a specific token type in the
input. When using `storeIn`, the string value of these tokens will be stored.

?> If you are using `by reflective()` to automatically get a declaration, then
the name of the stored elements must match the name of the constructor's
arguments.

For a full list of expectations, check the [reference](reference.md)

## A full example

Okay, let's put all of that into a full lexer + parser example.

```kotlin
// We assume that our token types are already defined elsewhere

data class Assignment(val element: String, val value: String) {
    companion object : PangoroNodeDeclaration<MyNode> by reflective()
}

val parser = pangoro {
    Assignment root {
        expect(element) storeIn "element"
        expect(opAssign)
        either {
            expect(stringValue) storeIn "value"
        } or {
            expect(intValue) storeIn "value"
        }
        expect(semicolon)
    }
}
val assignment = parser.parse(tokens) // tokens is our list of tokens
```

Notice that we added a `root` next to `Assignment`. This is because Pangoro
needs to know where to start (in other words, *what the root node of the AST
is*). We specify that by adding `root` next to our root node.

## Expecting other nodes

The expectation system also allows you to expect another node instead of just a
token type.

Let's assume that we have the following tokens

* `myVar` (`element`), `=` (`opAssign`), `"Hello!"` (`stringValue`), `;`
  (`semicolon`)
* `myOtherVar` (`element`), `=` (`opAssign`), `31` (`intValue`), `;`
  (`semicolon`)

This gives us a list of 8 tokens. Let's make a node that accepts two
assignments: a "first" assignment and a "second" assignment.

```kotlin
// Same as before
data class Assignment(val element: String, val value: String) {
    companion object : PangoroNodeDeclaration<MyNode> by reflective()
}

// That's new
data class TwoAssignments(val first: Assignment, val second: Assignment) {
    companion object : PangoroNodeDeclaration<TwoAssignments> by reflective()
}

val parser = pangoro {
    Assignment {
        // Same as before
        expect(element) storeIn "element"
        expect(opAssign)
        either {
            expect(stringValue) storeIn "value"
        } or {
            expect(intValue) storeIn "value"
        }
        expect(semicolon)
    }

    // Our new root, now expecting two assignments
    TwoAssignemnts root {
        expect(Assignment) storeIn "first"
        expect(Assignment) storeIn "second"
    }
}
val twoAssign = parser.parse(tokens)
/*  twoAssign == TwoAssignments(
        first = Assignment(element = myVar, value = "Hello!"),
        second = Assignment(element = myOtherVar, value = 31)
    )
*/
```

Now, that looks more like a tree!
