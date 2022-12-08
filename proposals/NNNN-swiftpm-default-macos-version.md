# Default macOS target version

* Proposal: [SE-NNNN](NNNN-swiftpm-default-macos-version.md)
* Authors: [Keith Smiley](https://github.com/keith)
* Review Manager: TBD
* Status: **Awaiting implementation**
* Implementation: [apple/swift-package-manager#5962](https://github.com/apple/swift-package-manager/pull/5962)

## Introduction

Currently Swift Package Manager packages that don't explicitly define
minimum supported platform versions automatically default to the first
supported version, which for many platforms is a version from 2017. This
adds friction for newly generated packages in some cases.

Swift-evolution thread: [Discussion thread topic for that
proposal](https://forums.swift.org/t/pitch-generate-package-swift-files-with-newer-macos-versions/61925)

## Motivation

Currently when building a brand new Swift Package Manager package on
macOS, the build targets macOS 10.13 from 2017. This can be a
frustrating new user experience when they add a dependency such as
`swift-syntax` that has a higher minimum supported version causing them
to immediately have to figure out how to configure this option.

## Proposed solution

I propose that we automatically add the minimum supported version for
new packages generated on macOS to the user's current macOS version. For
example instead of generating this `Package.swift` contents:

```swift
let package = Package(
    name: "name",
    ...
)
```

Today on macOS we would generate this contents:

```swift
let package = Package(
    name: "name",
    platforms: [.macOS("13.0")],
    ...
)
```

This makes the earlier build experience easier when adding other
packages, and easily allows developers to reduce this version if needed.

## Detailed design

In the case that the user is generating a new package on macOS, we fetch
the current macOS version, and include that in the `Package.swift`
template. Currently this ignores any non-macOS platform, but if there
were reasonable defaults we could provide for those as well, adding them
would be relatively trivial.

## Security

Not really.

## Impact on existing packages

No impact as this only affects newly generated packages.

## Alternatives considered

- Instead of generating this version we could just implicitly default to
  the current macOS version if no `platforms` were defined (example
  implementation
  [here](https://github.com/apple/swift-package-manager/pull/5904)) but
  this would affect existing packages, and may be unexpected.
- Only default to the current major version, excluding the minor
  version, in many cases this would likely be fine as well
