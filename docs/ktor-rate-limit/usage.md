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

## Reverse proxies and IP address spoofing

!> **Read this section carefully if you are using your Ktor server behind a 
reverse proxy.**

**By default, ktor-rate-limit uses the remote host's IP address to determine the
caller key. This ranges from potentially problematic to being a DoS vector.**

* If you **use a reverse proxy without doing anything in Ktor**, the remote 
  host will always be the IP address of your reverse proxy, making the rate 
  limit the same for everyone, which is obviously a bad thing.

Reverse proxy support is
[available in Ktor](https://ktor.io/servers/features/forward-headers.html). Use
it!

**Make sure you are using the correct feature.** `ForwadedHeaderSupport` is
  for the `Forwarded` header, `XForwardedHeaderSupport` is for the
  `X-Forwarded-Host`

**Your reverse proxy MUST NOT RETAIN THE FORWARDED HEADERS THE CLIENT SENDS**.
Forwarded headers always work in some kind of a list, typically like
`client, proxy 1, proxy 2`. Most proxies, if you are using their default
behavior, may only *append the host at the end of the existing header*, which
MUST NOT happen if the header is provided by the client, because every server
assumes that the client's IP address is the first one in the list.

Let's say the client's IP address is `2.2.2.2`. Let's take this request:

```
GET / HTTP/1.0
X-Forwarded-For: 99.99.99.99
```

A **misconfigured** reverse proxy would only *add the real IP address to the 
list*, and your server would receive this:

```
GET / HTTP/1.0
Host: realhost.com
X-Forwarded-For: 99.99.99.99, 2.2.2.2
...
```

(There may be other proxies at the end of the list if your server is behind
multiple proxies)

Ktor (and thus `ktor-rate-limit`) will give `99.99.99.99` as the real IP address
of the client, **which is not the case**. This could lead to DoS attacks by
flooding the server with different IP addresses (the rate limiting feature's
internal map would run out of memory), make rate limiting useless, or other very
bad things.

A **correctly configured** reverse proxy replaces this header entirely and
ignores the client's header value:

```
GET / HTTP/1.0
Host: realhost.com
X-Forwarded-For: 2.2.2.2
```

So:

* If your server is behind a single proxy, make sure that proxy correctly
  overwrites whatever the client sends, especially in the `X-Forwarded-Host`
  header.
* If you have multiple proxies (Clients -> P1 -> P2 -> P3 -> Ktor server), make
  sure the "front" proxy (that receives the clients' request, P1 here) and
  **only the front one**, otherwise you go back to the first issue, where only
  P1's IP address will ever show up.

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