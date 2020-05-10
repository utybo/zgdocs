# Using ktor-rate-limit

?> Have questions or issues? Feel free to open an issue on the
[bug tracker](https://github.com/utybo/ktor-rate-limit/issues). Also remember to
read the feature's KDoc comments, they have a lot of information too!

## Getting the dependency

`ktor-rate-limit` is distributed via JitPack. Visit
[this page](https://jitpack.io/#guru.zoroark/ktor-rate-limit) for more
information.

Typical installation in Gradle projects looks like this:

```
repositories {
    // other repositories, like mavenCentral()
    maven { url 'https://jitpack.io' }
}

dependencies {
    // other dependencies
    implementation 'guru.zoroark:ktor-rate-limit:VERSION'
}
```

Where `VERSION` is the version you want. It can also be many other things, such
as a commit hash. See the JitPack page for more information.

?> ktor-rate-limit uses SLF4J for logging under the `guru.zoroark.ratelimit`
logger (and `guru.zoroark.ratelimit.inmemory` for the built-in in-memory
storage)

## Feature installation

First, you will need to install `ktor-rate-limit` like any other feature.

```kotlin
install(RateLimit)
```

**The configuration block is optional.** Here are the defaults if you do not 
use a configuration block:

- `limit`: 50 requests
- `timeBeforeReset`: 2 minutes
- `callerKeyProducer`: The remote host's IP address
- `limiter`: The default in-memory implementation

You can configure each of these parameters by providing a block. Note that these
are global values that can be overridden on each rate-limited route. What you
configure when installing the feature is called the **global configuration**.

```kotlin
install(RateLimit) {
    limit = ...
    timeBeforeReset = ...
    callerKeyProducer = { ... }
    limiter = ...
}
```

For more information about what each parameter is, check the "parameters" 
section below

## Applying rate limiting

By default, the rate limit feature **does nothing**. You have to explicitly 
enable rate limiting on the routes you want to protect.

This is done simply by scoping all of these routes with a `rateLimited { ... }`
call. You can have any number of routes or methods inside of this block.

```kotlin
// Anything here will not be protected

rateLimited {
    // Anything here will be protected
}
```

All of the routes inside the block **will share the same rate limit "credit"**,
also known as a "bucket". See below for how this bucket is generated.

!> Make sure to not rate limit things that are already rate limited, e.g. avoid
using `rateLimited` twice on the same route. This could have unintended
consequences.

By default, `rateLimited` will use the feature's global configuration. You can
override some parameters here: `limit`, `timeBeforeReset`,
`additionalKeyExtractor`.

## How it works

Before going into each parameter, it's worth pausing to understand what each
component does.

The rate limit feature uses **buckets**. A single bucket can be seen as a
"request credit" and, if multiple routes use the same bucket, they will decrease
the same credit: the routes share the same rate limit.

Each bucket has an identifier, which is automatically generated. It is generated with three subkeys:

* A **caller key**, which is unique to the requester. The generator for this key
  is defined in the [feature's configuration](#feature-installation)
  (`callerKeyProducer`). By default, the remote host's IP address is used.
* A **routing key**, which is unique to the `rateLimited` call. This one cannot
  be changed and is randomly generated. This means that, by default, **different
  `rateLimited` blocks will each have their own bucket**.
* An **additional key**, which can be useful to differentiate calls that go in
  the same `rateLimited` block. This can be used to implement Discord's "primary
  parameter construct.

These three keys are then combined together, hashed (with the SHA1 algorithm)
and converted to Base64: this gives us our bucket identifier.

Each rate limit bucket keeps information about the current state. This inforamtion
contains:

- The number of remaining requests
- When the bucket "expires": that is, when the bucket gets reset

This state is managed by the **limiter**. A sane default implementation is
provided. This implementation stores the buckets in memory and automatically
purges itself with old keys to reduce the memory footprint. You can also 
implement your own limiter that uses some other form a storage. Just remember 
that this storage *must* be fast.

## Parameters

Here is a recap of the different parameters you can set as global configuration
(**G**, in the feature's configuration block) or as a route's configuration
(**R**, in the call to `rateLimited`).

* **G/R** `limit`: The maximum number of requests for the bucket.
* **G/R** `timeBeforeReset`: The amount of times before the bucket expires.
* **G** `limiter`: Implementation of the bucket storage.
* **G** `callerKeyProducer`: A function that creates a caller key based on a 
  given application call.
* **R** `additionalKeyExtractor`: A function that creates an additional key 
  based on a given application call.

?> Route configurations always override the global configuration.