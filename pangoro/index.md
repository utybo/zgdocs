# Pangoro

[Pangoro](https://github.com/utybo/Pangoro) is a parser for Lixy tokens.

A parser takes a list of tokens and turns them into an AST (Abstract Syntax
Tree). Pangoro allows you to do this with a clean, simple and nicely typed DSL.

Pangoro is available on Kotlin/JVM and Kotlin/JS, and is compatible with Kotlin
MPP.

While Pangoro works directly with Lixy, you can also use your own lexer and turn
its tokens into a list of `LixyToken` objects.

To get started, visit [this page](start).

## Example

```kotlin
// Types used by the parser (PangoroNode) and lexer (LixyTokenType)
data class Number(val value: String) : PangoroNode {
    companion object : PangoroNodeDeclaration<Number> by reflective()
}

data class Sum(val first: Number, val second: Number) : PangoroNode {
    companion object : PangoroNodeDeclaration<Addition> by reflective()
}

enum class Tokens : LinkTokenType {
    Number, Plus
}

// Lexer (from Lixy)
val lexer = lixy {
    state {
        matches("\\d+") isToken Tokens.Number
        "+" isToken Tokens.Plus
        " ".ignore
    }
}    
               
// Parser (from Pangoro)
val parser = pangoro {
    Number {
        expect(Tokens.Number) storeIn "value"
    }
    Sum root {
        expect(Number) storeIn "first"
        expect(Tokens.Plus)
        expect(Number) storeIn "second"
    }
}
val tokens = lexer.tokenize("123 + 4567")
val ast: Sum = parser.parse(tokens)
/* 
 * ast = Sum(
 *     first = Number(value = "123"),
       second = Number(value = "4567")
 * )
 */
```