# Getting started

## Downloading

Pangoro is published on JitPack. There are no released versions at the moment:
instead, use the commit hash as the version string.

[Check out Pangoro on JitPack](https://jitpack.io/#guru.zoroark/Pangoro).

Basically, you would add Pangoro to your project like this:

```
repositories {
    // your other repositories...
    maven { url 'https://jitpack.io' }
}

dependencies {
    implementation 'guru.zoraork.pangoro:VARIANT:VERSION'
}
```

Pangoro has a dependency on Lixy as well. Lixy is automatically pulled with
Pangoro.

Replace `VARIANT` with:

- For Kotlin/JVM, `pangoro-jvm`
- For Kotlin/JS, `pangoro-js`
- For Kotlin MPP, `pangoro` in the common dependencies. The platform-specific
  variants are retrieved automatically. If you only want to use Pangoro on a
  specific platform, add that platform's variant in the platform's dependencies.

Replace `VERSION` with the version you want.

## Using

You can use Pangoro anywhere in your code (as long as you declared a dependency
on Pangoro). For usage instructions, visit the [tutorial](tutorial)