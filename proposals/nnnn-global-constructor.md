# `@globalConstructor`

* Proposal: [SE-NNNN](nnnn-global-constructor.md)
* Authors: [Keith Smiley](https://github.com/keith)
* Review Manager: TBD
* Status: **Awaiting implementation**
* Implementation: [apple/swift#62428](https://github.com/apple/swift/pull/62428) or [apple/swift-syntax#1120](https://github.com/apple/swift-syntax/pull/1120)

## Introduction

The `@globalConstructor` attribute provides a way for functions to be
called automatically when an executable is launched, or a dynamic
library is loaded.

Swift-evolution thread: [Pitch](https://forums.swift.org/t/pitch-globalconstructor/61901)

## Motivation

There are many use cases where you may want to execute code without
having to directly call it. For example:

- Debugging where you'd like to insert a library into your application
  without having to rebuild your application manually calling the
  library
- Plugin systems where included optional libraries register themselves
  with the main application to change behavior
- Third party library authors who want to do 1 time setup on launch,
  without requiring developers to explicitly call their configuration

Today in these cases developers have to fallback to Objective-C/C++/C
(using `__attribute__((constructor))`), even if they only need to call a
Swift function.

## Proposed solution

Add a new `@globalConstructor` attribute that can be added to top-level
functions so that they are called automatically when the executable is
launched, or the dynamic library containing the function is loaded.

For example:

```swift
@globalConstructor
func foo() {
  print("This happens when the executable launches!")
}
```

In the rare case there are multiple global constructors that need to be
called in a specific order, they can be prioritized by providing a
custom priority:

```swift
@globalConstructor(priority: 42)
func first() {
  print("This happens first")
}

@globalConstructor(priority: 100)
func second() {
  print("This happens after first()")
}

@globalConstructor(priority: 1000)
func third() {
  print("This happens last")
}
```

## Detailed design

The implementation of this new attribute utilizes llvm's
[`global_ctors`](https://llvm.org/docs/LangRef.html#the-llvm-global-ctors-global-variable)
variable, which automatically adds the function to the platform specific
location for this functionality. This is the same behavior for the
implementation of `__attribute__((constructor))` in clang. Using this
imposes some behavior on the feature as far as disambiguation:

1. Constructors are called in ascending order (lowest numbers to highest
   numbers)
2. Constructors are only ordered in a single translation unit
3. Constructors without a priority are given a default priority
   (currently the maximum constructor priority number so those will be
   called last), and the order of constructors in the same translation
   unit with the same priority is undefined (today this undefined
   behavior is based on the order they are defined in the translation
   unit).
4. Theoretically some numbers (~0-100) are reserved for the standard
   library, but clang does not document, enforce, or even warn about
   this, and it doesn't seem like they are likely to conflict in the
   same translation unit, so I chose to mirror clangs behavior and
   ignore this.

The other major portion of the implementation is about restricting where
the attribute can be placed. Currently it can only be placed on top
level functions with the type `() -> Void`, all other types of functions
are disallowed as well as those with some other attributes like `async`
and `throws`. This could likely be loosened if there was a good reason
for it, but I imagine these limitations are acceptable for the majority
of use cases.

## Source compatibility

No impact as `@globalConstructor` is entirely additive.

## Effect on ABI stability

No impact. A function marked with `@globalConstructor` affects
containing binary, but consumers are not aware of this and can call the
function as normal if it's `public`.

## Effect on API resilience

No impact as `@globalConstructor` is entirely additive.

## Alternatives considered

- Various different spellings, `@globalInitializer`,
  `@moduleConstructor`, `@globalInit`, etc
- Not accepting any `priority` attribute as it's likely rare (and likely
  discouraged) to write many global initializers that depend on the
  order of other initializers to run successfully, but in this case you
  could not ever disambiguate especially in the case of LTO.
- Do nothing. Continue requiring that users fallback to
  Objective-C/C++/C to get this behavior
