# Swift Emit-Module Bug: ~Copyable Constraint Propagation Failure with Sequence Conformance

## Swift Issue

Filed as [swiftlang/swift#86669](https://github.com/swiftlang/swift/issues/86669)

## Description

The compiler fails during module emission (`-emit-module`) with the error "type 'Element' does not conform to protocol 'Copyable'" when a generic type with compound constraint (`Element: ~Copyable & Protocol`) has a nested type containing `UnsafeMutablePointer<Element>`, combined with conditional `Sequence` conformance and `borrowing Element` closures in extension files.

**Note**: Single constraint (`Element: ~Copyable`) does NOT trigger this bug. Only compound constraints do.

## Environment

- **Swift version**: 6.2.3 (swiftlang-6.2.3.3.21 clang-1700.3.137.100)
- **Target**: arm64-apple-macosx26.0
- **Flags**: `-enable-experimental-feature Lifetimes`

## Minimal Reproduction (60 lines)

```swift
public protocol Ordering: ~Copyable {
    static func isLessThan(_ lhs: borrowing Self, _ rhs: borrowing Self) -> Bool
}

@safe
public struct Container<Element: ~Copyable & Ordering>: ~Copyable {
    @usableFromInline final class Storage: ManagedBuffer<Int, Element> { ... }

    @safe
    public struct Bounded: ~Copyable {
        @usableFromInline var _storage: Storage
        @usableFromInline var _cachedPtr: UnsafeMutablePointer<Element>  // ERROR HERE
        public let capacity: Int
    }
}

extension Container.Bounded: Copyable where Element: Copyable {}

// TRIGGER: Sequence conformance
extension Container.Bounded: Sequence where Element: Copyable {
    public func makeIterator() -> AnyIterator<Element> { ... }
}
```

In separate extension file:
```swift
// TRIGGER: borrowing Element closure
extension Container.Bounded where Element: ~Copyable {
    public func withMin<R>(_ body: (borrowing Element) -> R) -> R? { ... }
}
```

## To Reproduce

```bash
git clone https://github.com/coenttb/swift-issue-emit-module-noncopyable-sequence
cd swift-issue-emit-module-noncopyable-sequence
swift build  # Fails with error
```

Or directly:
```bash
swiftc -swift-version 6 -enable-experimental-feature Lifetimes -emit-module \
  -module-name EmitModuleBug Sources/EmitModuleBug/*.swift
```

## Error Output

```
error: emit-module command failed with exit code 1 (use -v to see invocation)
Sources/EmitModuleBug/Container.swift:74:46: error: type 'Element' does not conform to protocol 'Copyable'
        var _cachedPtr: UnsafeMutablePointer<Element>
                                             `- error: type 'Element' does not conform to protocol 'Copyable'
```

## Conditions Required

All 6 conditions must be present to trigger the bug:

| # | Condition | Description |
|---|-----------|-------------|
| 1 | Compound generic constraint | `Element: ~Copyable & Protocol` (single constraint works) |
| 2 | Nested type with unsafe pointer | `UnsafeMutablePointer<Element>` stored property |
| 3 | Conditional Sequence conformance | `extension Type: Sequence where Element: Copyable` |
| 4 | Extension file with borrowing closure | `(borrowing Element) -> R` in separate .swift file |
| 5 | Library target | Uses `-emit-module` flag |
| 6 | Lifetimes feature | `-enable-experimental-feature Lifetimes` |

## Verified Test Results

| Test | Description | Result |
|------|-------------|--------|
| Parse only | `swiftc -parse *.swift` | ✅ Compiles |
| Single constraint | `Element: ~Copyable` (no protocol) | ✅ Compiles |
| Custom protocol | Non-Sequence conditional conformance | ✅ Compiles |
| Same-file borrowing | `borrowing Element` in main file | ✅ Compiles |
| Emit module | `swiftc -emit-module *.swift` | ❌ Fails |

**Key finding**: The bug is specific to the `-emit-module` compilation phase and requires the exact combination of Sequence conformance with borrowing closures in extension files.

## Workaround

Disable `Sequence` conformance and provide `forEach(_:)` as an alternative:

```swift
// Instead of: for element in bounded { ... }
bounded.forEach { element in ... }
```

Moving `borrowing Element` methods to the main type file works for simple cases but does not work for complex real-world codebases.

## Impact

This bug blocks `Sequence` conformance for all generic data structures that:
- Use compound constraints with `~Copyable`
- Have nested types with unsafe pointers
- Use `borrowing Element` closures in extension files

This affects the Swift Primitives project's Heap implementation and similar move-only data structures, preventing standard iteration patterns (`for-in` loops, `map`, `filter`, etc.).

## Related Issues

Potentially related to ~Copyable constraint propagation in module interface generation.
