# Pangoro Reference

This page lists a reference of all of the blocks in Pangoro you can use to build
your parser.

## Expectations

### Token expectation

Expects a token of a specific type to be present in the input.

```kotlin
// Regular syntax
expect(tokenType)
// Short syntax
+tokenType
```

* `tokenType` is a `LixyTokenType` and is the expected type

This expectation's result can be stored and is of type `String`.

### Node expectation

Expects a node to be present at that point. The node's description is used to
try and parse the tokens.

```kotlin
// Regular syntax
expect(MyNode)
// Short syntax
+MyNode
```

* `MyNode` is a `PangoroNodeDeclaration<T>` where `T` is the expected node.
  Works best if you are using a `companion object`.

This expectation's result can be stored and is of type `T`.

### Either expectation

This expectation is a branching expectation. It gives an "or"-type construct,
where at this point any of the branches are possible in the input.

The branches are tried in the order they are declared. The first successful
branch's result is taken, and the following branches are not checked.

```kotlin
either {
    // ...expectations...
} or {
    // ...expectations...
} or {
    // ...expectations...
} // ...
```

This expectation does not have any result, but transmits all of what was stored
in each branch, so you could for example have every branch store something
somewhere.

```kotlin
either {
    // ...
    expect(...) storeIn "hello"
} or {
    // ...
    expect(...) storeIn "hello"
}
```

With this expectation, we're guaranteed to have a value stored with the name
"hello" in every branch.