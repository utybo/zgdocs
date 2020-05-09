# Getting started

## Downloading

Lixy is published on JitPack. There are no released versions at the moment: instead, use the commit hash as the version string.

[Check out Lixy on JitPack](https://jitpack.io/#guru.zoroark/lixy).

Basically, you would add Lixy to your project like this:

```
repositories {
    // your other repositories...
    maven { url 'https://jitpack.io' }
}

dependencies {
    implementation 'guru.zoraork.lixy:VARIANT:VERSION'
}
```

Replace `VARIANT` with:

- For Kotlin/JVM, `lixy-jvm`
- For Kotlin/JS, `lixy-js`
- For Kotlin MPP, `lixy` in the common dependencies. The platform-specific variants are retrieved automatically. If you only want to use Lixy on  a specific platform, add that platform's variant in the platform's dependencies.

Replace `VERSION` with the version you want.

## Using

You can use Lixy anywhere in your code (as long as you declared a dependency on
Lixy). For usage instructions, visit the [tutorial](tutorial)