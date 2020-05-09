# Lixy

[**Lixy**](https://github.com/utybo/Lixy) is a lexer library for creating lexers
on Kotlin, the easy way. It does not use any code generation of any kind,
preferring a simpler approach, entirely in Kotlin.

Lixy is available on Kotlin/JVM and Kotlin/JS and is compatible with Kotlin MPP.

To get started, visit [this page](start).

## Lexers

A [lexer](https://en.wikipedia.org/wiki/Lexical_analysis) is a tool that turns a
string of characters into a sequence of tokens. This is extremely useful for
things like *compilers*: when using a lexer first, you no longer need to deal
with individual characters, you will deal with full sequences.

What's more, with more advanced lexers, you can also immediately isolate some
special constructs, like string and character escaping.

## Example

Lixy is a multi-state lexer with a very intuitive DSL:

```kotlin
// We need this so we can use e.g. DOT instead of MyTokenTypes.DOT
import MyTokenTypes.* 

enum class MyTokenTypes : LixyTokenType {
    DOT, WORD, WHITESPACE
}

val lexer = lixy {
    state {
        "." isToken DOT
        anyOf(" ", "\n", "\t") isToken WHITESPACE
        matches("[A-Za-z]+") isToken WORD
    }
}

val tokens = lexer.tokenize("Hello Kotlin.\n")
/* 
 * tokens = [
 *      ("Hello", 0, 5, WORD), 
 *      (" ", 5, 6, WHITESPACE), 
 *      ("Kotlin", 6, 11, WORD),
 *      (".", 11, 12, DOT),
 *      ("\n", 12, 13, WHITESPACE)
 *  ]
 */
```

Everything you see above is Kotlin code: Lixy uses a "DSL" (domain-specific
language), here a "language in a language" that uses some of Kotlin's features
to look different and allow you to do complicated things the easy way.

## Doing more

Lixy provides a fairly uncomplicated list of tokens, which can be processed by
anything. [Pangoro](/pangoro/) is a parser specifically intended for use with
Lixy.