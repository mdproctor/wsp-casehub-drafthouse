---
layout: post
title: "The config default that didn't need a PostConstruct"
date: 2026-06-06
type: phase-update
entry_type: note
subtype: diary
projects: [drafthouse]
tags: [quarkus, smallrye-config, config-mapping, mockito, testing]
---

The session started with a clear goal: close two small issues. One was removing an annotation from a test class. The other was making `ReviewSessionService`'s storage path configurable instead of hardcoded to `~/.drafthouse/reviews/`. Neither looked hard going in.

The annotation removal was trivial — `@QuarkusTest` on a class that instantiates everything with `new`, no CDI in sight, no lifecycle. Gone in two lines.

The storage path issue was more interesting. My first instinct was the obvious path: make the field `Optional<String>`, inject a `@ConfigProperty`, and resolve the `user.home` system property in a `@PostConstruct` method. The reasoning was that `@WithDefault` in `@ConfigMapping` would handle simple string literals, but system property expansion is something you'd normally handle in code — because `@ConfigProperty(defaultValue = "...")` is explicitly excluded from expression expansion by the MicroProfile spec.

That assumption was wrong, in a useful way.

`@WithDefault` in a `@ConfigMapping` interface is *not* `@ConfigProperty(defaultValue = "...")`. They look similar but go through different machinery. SmallRye Config's expression interceptor runs before type conversion — so `@WithDefault("${user.home}/.drafthouse/reviews")` on a `Path` return type expands `${user.home}` from the System Properties ConfigSource (ordinal 400) and then converts the resulting string to a `Path`. No `@PostConstruct`, no mutable instance field, no `Optional<String>`, no `Path.of()` at the call site.

The design that I'd sketched with six moving parts collapsed to one interface method:

```java
interface Storage {
    @WithDefault("${user.home}/.drafthouse/reviews")
    Path root();
}
```

There was a naming trap in the same area. The spec originally used `${sys:user.home}` — a prefix that looks like a system-property handler, common in Spring and other config frameworks. In SmallRye Config, the colon is the *fallback-value separator*: `${sys:user.home}` means "expand property named `sys`; if absent, use literal `user.home`". The double-colon syntax `${sys::user.home}` invokes a named handler, but handler availability is version-specific. The plain `${user.home}` is what you want. Easy to get wrong because the single-colon version produces a wrong path with no error.

The restructuring of `DraftHouseConfig` from a flat interface to nested sub-interfaces (`Reviewer`, `Storage`) introduced a test failure that took a moment to understand. `when(config.reviewer().personality()).thenReturn(...)` threw `NullPointerException` — in `setUp()`, not in a test method. The chain `config.reviewer()` returns null on a plain mock before the stub is registered, so `.personality()` on null explodes during Mockito's own setup machinery. The fix is to stub the intermediate first: `when(config.reviewer()).thenReturn(reviewerMock)`, then stub the leaf. Using `RETURNS_DEEP_STUBS` works too but hides unstubbed leaves that silently return zero or null later.

While the test suite ran, one test failed intermittently: `ReviewSessionLifecycleTest.query_dispatchesSanitizedDecline_...`. The Qhorus channel name validation requires each segment to match `[a-z][a-z0-9]*(-[a-z0-9]+)*` — must start with a letter. `DraftHouseMcpTools` uses `UUID.randomUUID().toString()` as the channel slug. A UUID starting with a digit (which happens ~62.5% of the time) fails the validation. Filed as #35. The fix — prefix the slug with a letter — is one line.
