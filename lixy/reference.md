# Reference

This reference contains all of the recognizers and matchers available in Lixy.

## Recognizers

This section describes all of the available recognizers.

Recognizers can always be used as-is in the DSL. Recognizers by themselves don't
do anything: you have to pair them with a matching pattern (`isToken` or
`.ignore`)

### Recognizer types

#### Recognizers (R)

Recognizers are objects of the type `LixyTokenRecognizer`. They are the simplest
type of recognizer and are usually built using a function call (e.g.
`anyOf(...)`) or are automatically built from pseudo-recognizers.

#### Pseudo-recognizers (PR)

Pseudo-recognizers are objects that are *not inherently recognizers* (i.e. they
don't implement the `LixyTokenRecognizer` interface) yet are understood as such.
This is done for the user's convenience and they can only be used as-is with the
DSL: the library's functions expect recognizer objects.

The conversion between pseudo-recognizers and regular recognizers is done by the
`guru.zoroark.lixy.matchers.toRecognizer` function in the
[LixyTokenRecognizer.kt](https://github.com/utybo/Lixy/blob/master/src/commonMain/kotlin/guru/zoroark/lixy/matchers/LixyTokenRecognizer.kt)
file.

#### Modifier (M)

A modifier takes an existing recognizer and modifies it in some way, creating a
new recognizer.

### String (PR)

A `String` is a pseudo-recognizer that is automatically turned into a
`LinkStringTokenRecognizer`.

This recognizer matches against the entire string and is case sensitive. For
example `"World" isToken myTokenType` would match against `World` but not
`world`.

Example:

```kotlin
val tHello = tokenType()
val tGoodbye = tokenType()
val lexer = lixy {
    state {
        "Hello" isToken tHello
        "Goodbye" isToken tGoodbye
        " ".ignore
    }
}
val tokens = lexer.tokenize("Hello Goodbye Hello")
// tokens = [Hello{tHello},Goodbye{tGoodbye},Hello{tHello}]
```

### Character range (PR)

Character ranges (`CharRange`, usually constructed with `'a'..'z'`) are
pseudo-recognizers that are automatically turned into a
`LixyCharRangeTokenRecognizer`.

This recognizer matches against a single character that is present in the range.
For example `'a'..'c' isToken myTokenType` would match *two separate tokens* in
`ba` or `cb`.

The range is taken as-is and thus is case sensitive.

Example:

```kotlin
val tac = tokenType()
val tdf = tokenType()
val lexer = lixy {
    state {
        'a'..'c' isToken tac
        'd'..'f' isToken tdf
        ('A'..'Z').ignore
    }
}
val tokens = lexer.tokenize("afAXebc")
// tokens = [a{tac},f{tdf},e{tdf},b{tac},c{tac}]
```

### anyOf set recognizer (R)

The `anyOf` recognizer one of the strings from a set. It is case-sensitive.

Creating a recognizer is simple, and is simply done by using

```
anyOf("string 1", "other string", "sTrInG tHrEe")
```

You can place as many strings as you want in the `anyOf` call, but you must have
at least one.

Internally, this recognizer is optimized to check its list as little as
possible, so you can place as many strings as you want in there.

If you want to reuse the same list over multiple recognizers/states/whatever, you can do it like so:

```kotlin
val myStrings = listOf("Hello", "Hey", "Hi").toTypedArray()

// Then, wherever you want to create the recognizer
anyOf(*myStrings) ...
```

This uses Kotlin's spread operator for turning an array into the `vararg`
arguments of a function.

Example:

```kotlin
val thello = tokenType()
val tbonjour = tokenType()
val lexer = lixy {
    state {
        anyOf("Hello", "Hi") isToken thello
        anyOf("Bonjour", "Salut") isToken tbonjour
        " ".ignore
    }
}
val tokens = lexer.tokenize("Hello Bonjour Hi")
// tokens = [Hello{thello},Bonjour{tbonjour},Hi{thello}]
```

### Regex matches recognizer (R)

The Regex recognizer, built using the `matches` function, recognizes patterns
using a regular expression.

!> Kotlin/JS support for `matches` is not optimized and might be very expensive
on long input strings.

Simply uses `matches("regex expression")` to create a recognizer.

Example:

```kotlin
val tNumber = tokenType()
val tWord = tokenType()
val lexer = lixy {
    state {
        matches("\\d+") isToken tNumber
        matches("\\w+") isToken tWord
        " ".ignore
    }
}
val tokens = lexer.tokenize("Hello 42 World 09")
// tokens = [Hello{tWord},42{tNumber},World{tWord},09{tNumber}]
```

### Repeated recognizer (M)

A repeated recognizer is a modifier that takes a base recognizer and matches
against a repetition of that recognizer. The minimum and maximum amount of
repetitions can be customized.

There are three ways of creating a repeated recognizer, where `recognizer` is another (pseudo-)recognizer:

* `recognizer.repeated`: Match against a repetition with at least one element and no maximum
* `recognizer.repeated(x)`: Match against a repetition with at least `x` repetitions and no maximum
* `recognizer.repeated(x, y)`: Match against a repetition with at least `x` repetitions and at most `y` repetitions.

?> The last two are actually the same function, and can take named arguments.
The parameters are `min` (default 1) and `max` (default null to indicate no
maximum). So you could use `recognizer.repeated(max = 10)` for a repetition of at least 1 and at most 10 occurences.

Example:

```kotlin
val ta = tokenType()
val tb1 = tokenType()
val tb2 = tokenType()
val tc = tokenType()
val lexer = lixy {
    state {
        "a".repeated isToken ta
        "b".repeated(max = 3) isToken tb1
        "b".repeated(4, 6) isToken tb2
        "c".repeated(2) isToken tc
        
    }
}
val tokens = lexer.tokenize("abbaaaacccbbbb")
// tokens = [a{ta},bb{tb1},aaaa{ta},ccc{tc},bbbb{tb2}]
```

## Matchers

Recognizers only have the ability to recognize patterns but do not have any idea
of what to do with the pattern they spot.

Matchers match the result of recognizers to a specific behavior. There are
currently two behaviors

### Create a token (isToken)

`isToken` takes the recognized pattern and creates a token out of it using the
provided token type. The lexer then continues at the end of the recognized 
pattern.

### Ignore (.ignore)

`.ignore` takes the recognized pattern and ignores it -- very similar to
`isToken`, except that no token is actually created.

## States

A state is the set of rules the lexer currently follows. Lixy can work with
either one or multiple states.

### Single-state

Declaring a single-state parser is easy: you simply define a single `state`
block with nothing else attached to it.

```kotlin
val lexer = lixy {
    state {
        "Hello" isToken ...
    }
}
```

### Multi-state

A multi-state lexer has a few differences:

* It has multiple `state` block
* Each block *must* additionally be preceded by a "state label"
* One state *must* be defined as the default state: that is the state with which
  Lixy starts the lexing process

(TODO, create label with `stateLabel()` or with a dedicated enum, create the states with `myLabel state { ... }`)

### Default state

(TODO, just use `default` instead of the state label)

### Changing the state

(TODO, use `thenState myLabel` after the matcher)