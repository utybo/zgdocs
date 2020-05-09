# Tutorial/guide

Welcome to Lixy! This page will guide you through your first steps using the Lixy lexer.

?> For instructions on how to get Lixy, see [Getting Started](start.md).

## Lexers and you

Before getting started, we need to have a word about lexers.

A *lexer* is a tool commonly used for creating compilers and for understanding various programming languages. Its goal is to take a string of characters as its input and to output a list of *tokens*. A tokens is kind of an "annotated substring" of our input.

For example, out of the input `I ate an apple`, we could "extract" the following tokens:

* `I` of type "word", starting at index 0, finishes at index 1
* `ate` of type "word", starting at index 2, finishes at index 5
* `an` of type "word", starting at index 6, finishes at index 8
* `apple` of type "fruit", starting at index 9, finishes at index 14

In Lixy, tokens have information about what they represent ("I", "ate", "an", "apple"), their type ("word" or "fruit" here), and their location in the original string (here the starting and finishing index).

## The Lixy block

Lixy uses a DSL for pretty much everything. This means that your lexer will look very clean while still being entirely Kotlin code.

Constructing the lexer is 95% of the work, so we will spend most of our time talking about how this can be done.

Constructing the Lixy lexer is always done using a **Lixy block**.

```kotlin
val lexer = lixy {

}
```

This block will contain everything we need for our lexer. The `lixy` function will return our constructed lexer, which we then store in the `lexer` variable.

?> If the lexer construction fails (e.g. misconfigured lexer), a `LixyException` will be thrown.

## The single-state lexer

For now, we will work with a single state.

Lexers can be "multi-state", meaning that they have different sets of rules they can use and can jump back and forth between these sets of rules in different situations. For example, you might want to handle regular code and the content of a string differently: you would do that by using different states.

Let's only focus on having a single state, that is, a single set of rules. We can declare our single-state like this:

```kotlin
val lexer = lixy {
    state {
        // ...
    }
}
```

We can now define all of our rules inside this state.

## Recognizers and matchers

"Rules" in lixy are actually called matchers. Matchers themselves are made of
two thing: a recognizer and a token type. Each matcher produces a token.

The job of a recognizer is to look at our input a specific index, and say
whether or not it is matched by that recognizer. For example, we can have a
simple recognizer that checks if the character at that index is a dot `.`: if
yes, the recognizer "passes".

The recognizer by itself only recognizes patterns, it does not produce anything.
We need to pair the recognizer with a matching behavior: what do we do once the
pattern has been recognized? At the moment, there are two possibilities:

* Emit a token and move on
* Ignore the pattern and move on without emitting a token

We will focus about the first point first.

Typically, you would declare a matcher (in a state) like so:

```kotlin
   anyOf("hello", "bonjour", "hola") isToken Greeting
// --------------------------------- ################
```

The underlined part (`---`) defines the *recognizer*. These are just simple
functions that build a recognizer object. `isToken` is more interesting: it is a
function that both creates the matcher and adds it to the current state. The
matcher has `###` under it.

This matcher will recognize one of "hello", "bonjour" or "hola" and, if
recognized, emits a token of type Greeting.

A list of recognizers and matchers is available in the
[reference](reference.md).

## Token types

A quick word about token types and we'll create some lexers: each token has an
attached "token type".

In Lixy, token types are just instances of the `LixyTokenType` interface. It is
a simple marker interface that does not actually require you to implement
anything. There are two recommended ways of doing this:

* By calling `tokenType()`. You can also provide a name as a string argument
  (`tokenType("My type's name")`). This is fairly good for small examples but 
  not very scalable. You cannot attach special values to types either.
* By creating an enum class that will contain all your token types

```kotlin
enum class MyTypes : LixyTokenType {
    Number, Whitespace, Word, Something
}
```

You can of course also do anything else you think works best for you: the only
restriction is that token types need to implement the `LixyTokenType` interface.

## A simple lexer

Now that we know all of this, let's make a simple lexer:

```kotlin
val whitespace = tokenType()
val greeting = tokenType()
val lexer = lixy {
    anyOf("\t", "\n", " ") isToken whitespace
    "Hello" isToken greeting
}
val tokens = lexer.tokenize("Hello\n\tHello Hello")
```

The `tokens` variable receives a list of tokens:

* `Hello` (type `greeting`, starts at 0, ends at 5)
* `\n` (type `whitespace`, starts at 5, ends at 6)
* `\t` (type `whitespace`, starts at 6, ends at 7)
* `Hello` (type `greeting`, starts at 7, ends at 12)
* ` ` (type `whitespace`, starts at 12, ends at 13)
* `Hello` (type `greeting`, starts at 13, ends at 18)

## Multi-state lexers

For now, we have only used single-state lexers. While these are powerful enough for simple examples, it would be nice to have a more powerful solution.

A state is, simply put, a set of rules that Lixy follows. In a single-state
lexer, we only have a single set of rules, making it quite simple. In a
multi-state lexer, we would have multiple possible states, meaning that we could
be following a number of different states at any point.

Let's take the example of a simple string lexer. 

* Outside of a string, we can have numbers or bare words, as well as quotation
  marks. When we meet a quotation mark (`"`), it means that we have reached a
  string, and we need to follow a different set of rules.
* Inside a string, we have to produce a token for the entire content, *but* we
  also need to watch out for "escaped" characters (`\u01234` for unicode
  characters, `\"` for a quotation mark that is part of the string) and
  quotation marks that end the string.

Thus, we could have two different set of rules, so two states.

Before we jump into making this lexer with Lixy, we need a quick word on how we
identify these states.

## State labels

Much like tokens have a type, states have a label. A label simply identifies the
state and allows us to refer back to it in a simple way.

States labels can be defined using two methods, just like token types.

* By calling `stateLabel()`, which creates a label we can use immediately.
* By using an enum. Here, we implement the `LixyStateLabel` interface

```kotlin
enum class MyLabels : LixyStateLabel {
    InString, OutsideOfString
}
```

We use these labels by simply putting the label before the `state`.

```kotlin
// Using stateLabel()
val stateOne = stateLabel()
val stateTwo = stateLabel()
lixy {
    stateOne state {
        // ...
    }
    stateTwo state {
        // ...
    }
}

// Using an enum
lixy {
    MyLabels.InString state {
        // ...
    }
    MyLabels.OutsideOfString state {
        // ...
    }
}
```

## Default state

Lixy needs to know where we start, thus we must define a state as the "default
state". This can be done in two ways:

* By using a special label `default` which is automatically created by Lixy.

```kotlin
lixy {
    default state {
        // ...
    }
    MyLabels.InString state {
        // ...
    }
}
```

* By telling Lixy what the default state is

```kotlin
lixy {
    default state MyLabels.OutsideOfString
    // ...
}
```

## Moving between states

Moving between states can only been done upon a match. Simply put `thenState stateLabel` after a match, for example:

```kotlin
"hello" isToken greeting thenState hasBeenGreeted
```

In this example `greeting` is a token type and `hasBeenGreeted` is a state
label. When this matcher matches, the lexer will then use the `hasBeenGreeted`
state from now on.

## A more complex lexer

With all of this knowledge, let's make a multi-state lexer!

```kotlin
// States
val inString = stateLabel()
val outsideOfString = stateLabel()

// Token types
val number = tokenType("number")
val word = tokenType("word")
val stringStart = tokenType("string start")

val stringEnd = tokenType("string end")
val stringContent = tokenType("string content")
val unicodeChar = tokenType("unicode char")
val escapedQuote = tokenType("escaped quote")

val lexer = lixy {
    default state outsideOfString
    outsideOfString state {
        // Match a "
        "\"" isToken stringStart thenState inString
        ('a'..'z').repeated isToken word
        ('0'..'9').repeated isToken number
        " ".ignore
    }
    inString state {
        // Match a "
        "\"" isToken stringEnd thenState outsideOfString
        // Match a \"
        "\\\"" isToken escapedQuote
        // Match a \u followed by a number
        matches("""\\u\d+""") isToken unicodeChar
        // Match anything else that does not contain a quote or a backslash
        matches("""([^\"])""") isToken stringContent
    }
}
```

i> The """ is a raw string in Kotlin, which means we don't need to escape things
inside, but RegExes require us to escape `\`. Otherwise, we would need to write
`\\\\` so that Kotlin escapes them (resulting in a string that contains `\\`,
which would then be interpreted as matching a single backslash by Regex)
