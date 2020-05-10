# ktor-rate-limit

[**ktor-rate-limit**](https://github.com/utybo/ktor-rate-limit) is a flexible
and powerful rate limit feature for Ktor servers. It is licensed under the
Apache License 2.

## Rate limiting

Rate limiting is an important concept to protect yourself from DDoS attacks.
While it is not a universal solution, it allows you to protect APIs by limiting
how many times they are called.

`ktor-rate-limit` implements Discord's rate limit style for Ktor servers, and is
stupidly easy to use.

!> Discord's concept of "Global rates" is not implemented yet

For more information on how to use `ktor-rate-limit`, visit
[the usage page](usage.md).

## Example

```kotlin
install(RateLimit)

routing {
    route("/myroute) {
        // ...
    }

    rateLimited {
        // Everything in here is protected by the rate limit feature
        get {
            // ...
        }
        route(...) {
            // ...
        }
    }
}
``` 