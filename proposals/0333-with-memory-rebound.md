# Expand usability of `withMemoryRebound`

* Proposal: [SE-0333](0333-with-memory-rebound.md)
* Authors: [Guillaume Lessard](https://github.com/glessard), [Andrew Trick](https://github.com/atrick)
* Review Manager: [Ben Cohen](https://github.com/airspeedswift)
* Status: **Active review (November 30 - December 9, 2021)**
* Implementation: [draft pull request][draft-pr]
* Bugs: [SR-11082](https://bugs.swift.org/browse/SR-11082), [SR-11087](https://bugs.swift.org/browse/SR-11087)

[draft-pr]: https://github.com/apple/swift/pull/39529
[pitch-thread]: https://forums.swift.org/t/pitch-expand-usability-of-withmemoryrebound/52500

## Introduction

The function `withMemoryRebound(to:capacity:_ body:)`
executes a closure while temporarily binding a range of memory to a different type than the callee is bound to.
We propose to lift some notable limitations of `withMemoryRebound` and enable rebinding to a larger set of types,
as well as rebinding from raw memory pointers and buffers.

Swift-evolution thread: [Pitch thread][pitch-thread]

## Motivation

When using Swift in a systems programming context or using Swift with libraries written in C,
we occasionally need to temporarily access a range of memory as instances of a different type than has been declared
(the pointer's `Pointee` type parameter).
In those cases, `withMemoryRebound` is the tool to reach for,
allowing scoped access to the range of memory as another type.

As a reminder, the function is declared as follows on the type `UnsafePointer<Pointee>`:
```swift
func withMemoryRebound<T, Result>(
  to type: T.Type,
  capacity count: Int,
  _ body: (UnsafePointer<T>) throws -> Result
) rethrows -> Result
```

This function is currently more limited than necessary.
It requires that the stride of `Pointee` and `T` be equal.
This requirement makes many legitimate use cases technically illegal,
even though they could be supported by the compiler.

We propose to allow temporarily binding to a type `T` whose stride is
a whole fraction or whole multiple of `Pointee`'s stride,
when the starting address is properly aligned for type `T`.
As before, `T`'s memory layout must be compatible with that of`Pointee`.

<!-- 
Two types are mutually layout compatible if their in-memory representation has the same size and alignment
or they have the same number of mutually layout-compatible elements.
The concept of layout compatibility is not strictly defined, but here are examples of mutually layout compatible types:
- a type and its typealias
- two integer types of the same bit width
- class references, and `AnyObject` existentials
- two pointer types
- frozen structs with one stored property, and a value of the same type as the stored property

Types can also be layout compatible in a non-mutual manner; 
for example, homogeneous aggregates are layout compatible with larger homogeneous aggregates if their common elements are mutually layout compatible.
 -->

For example, suppose that a buffer of `Double` consisting of a series of (x,y) pairs is returned from data analysis code written in C.
The next step might be to display it in a preview graph, which needs to read `CGPoint` values.
We need to copy the `Double` values as pairs to values of type `CGPoint` (when executing on a 64-bit platform):

```swift
var count = 0
let pointer: UnsafePointer<Double> = calculation(&count)

var points = Array<CGPoint>(unsafeUninitializedCapacity: count/2) {
  buffer, initializedCount in
  var p = pointer
  for i in buffer.indices where p+1 < pointer+count {
    buffer.baseAddress!.advanced(by: i).initialize(to: CGPoint(x: p[0], y: p[1]))
    p += 2
  }
  initializedCount = pointer.distance(to: p)/2
}
```

We could do better with an improved version of `withMemoryRebound`.
Since `CGPoint` values consist of a pair of `CGFloat` values,
and `CGFloat` values are themselves layout-compatible with `Double` (when executing on a 64-bit platform):
```swift
var points = Array<CGPoint>(unsafeUninitializedCapacity: data.count/2) {
  buffer, initializedCount in
  pointer.withMemoryRebound(to: CGPoint.self, capacity: buffer.count) {
    buffer.baseAddress!.initialize(from: $0, count: buffer.count)
  }
  initializedCount = buffer.count
}
```

Alternately, the data could have been received as bytes from a network request, wrapped in a `Data` instance.
Previously we would have needed to do:
```swift
let data: Data = ...

var points = Array<CGPoint>(unsafeUninitializedCapacity: data.count/MemoryLayout<CGPoint>.stride) {
  buffer, initializedCount in
  data.withUnsafeBytes { data in
    var read = 0
    for i in buffer.indices where (read+2*MemoryLayout<CGFloat>.stride)<=data.count {
      let x = data.load(fromByteOffset: read, as: CGFloat.self)
      read += MemoryLayout<CGFloat>.stride
      let y = data.load(fromByteOffset: read, as: CGFloat.self)
      read += MemoryLayout<CGFloat>.stride
      buffer.baseAddress!.advanced(by: i).initialize(to: CGPoint(x: x, y: y))
    }
    initializedCount = read / MemoryLayout<CGPoint>.stride
  }
}
```

In this case having the ability to use `withMemoryRebound` with `UnsafeRawBuffer` improves readability in a similar manner as in the example above:

```swift
var points = Array<CGPoint>(unsafeUninitializedCapacity: data.count/MemoryLayout<CGPoint>.stride) {
  buffer, initializedCount in
  data.withUnsafeBytes {
    $0.withMemoryRebound(to: CGPoint.self) {
      (_, initializedCount) = buffer.initialize(from: $0)
    }
  }
}
```

## Proposed solution

We propose to lift the restriction that the strides of `T` and `Pointee` must be equal.
This means that it will now be considered correct to re-bind from a homogeneous aggregate type to the type of its constitutive elements,
as they are layout compatible, even though their stride is different.

### Instance methods of `UnsafePointer<Pointee>` and `UnsafeMutablePointer<Pointee>`

We propose to lift the restriction that the strides of `T` and `Pointee` must be equal, when calling `withMemoryRebound`.
The function declarations remain the same on these two types,
though given the relaxed restriction, 
we must clarify the meaning of the `capacity` argument.
`capacity` shall mean the number of strides of elements of the temporary type (`T`) to be temporarily bound.
The documentation will be updated to reflect the changed behaviour.
We will also add parameter labels to the closure type declaration to benefit code completion (a source compatible change.)

```swift
extension UnsafePointer {
  public func withMemoryRebound<T, Result>(
    to type: T.Type,
    capacity count: Int,
    _ body: (_ pointer: UnsafePointer<T>) throws -> Result
  ) rethrows -> Result
}

extension UnsafeMutablePointer {
  public func withMemoryRebound<T, Result>(
    to type: T.Type,
    capacity count: Int,
    _ body: (_ pointer: UnsafeMutablePointer<T>) throws -> Result
  ) rethrows -> Result
}
```

### Instance methods of `UnsafeRawPointer` and `UnsafeMutableRawPointer`

We propose adding a `withMemoryRebound` method, which currently does not exist on these types.
Since it operates on raw memory, this version of `withMemoryRebound` places no restriction on the temporary type (`T`).
It is therefore up to the program author to ensure type safety when using these methods.
As in the `UnsafePointer` case, `capacity` means the number of strides of elements of the temporary type (`T`) to be temporarily bound.

```swift
extension UnsafeRawPointer {
  public func withMemoryRebound<T, Result>(
    to type: T.Type,
    capacity count: Int,
    _ body: (_ pointer: UnsafePointer<T>) throws -> Result
  ) rethrows -> Result
}

extension UnsafeMutableRawPointer {
  public func withMemoryRebound<T, Result>(
    to type: T.Type,
    capacity count: Int,
    _ body: (_ pointer: UnsafeMutablePointer<T>) throws -> Result
  ) rethrows -> Result
}
```

### Instance methods of `UnsafeBufferPointer` and `UnsafeMutableBufferPointer`

We propose to lift the restriction that the strides of `T` and `Pointee` must be equal, when calling `withMemoryRebound`.
The function declarations remain the same on these two types.
The capacity of the buffer to the temporary type will be calculated using the length of the `UnsafeBufferPointer<Element>` and the stride of the temporary type.
The documentation will be updated to reflect the changed behaviour.
We will add parameter labels to the closure type declaration to benefit code completion (a source compatible change.)

```swift
extension UnsafeBufferPointer {
  public func withMemoryRebound<T, Result>(
    to type: T.Type,
    _ body: (_ buffer: UnsafeBufferPointer<T>) throws -> Result
  ) rethrows -> Result
}

extension UnsafeMutableBufferPointer {
  public func withMemoryRebound<T, Result>(
    to type: T.Type,
    _ body: (_ buffer: UnsafeMutableBufferPointer<T>) throws -> Result
  ) rethrows -> Result
}
```

### Instance methods of `UnsafeRawBufferPointer` and `UnsafeMutableRawBufferPointer`

We propose adding a `withMemoryRebound` method, which currently does not exist on these types.
Since it operates on raw memory, this version of `withMemoryRebound` places no restriction on the temporary type (`T`).
It is therefore up to the program author to ensure type safety when using these methods.
The capacity of the buffer to the temporary type will be calculated using the length of the `UnsafeRawBufferPointer` and the stride of the temporary type.

Finally the set, we propose to add an `assumingMemoryBound` function that calculates the capacity of the returned `UnsafeBufferPointer`.

```swift
extension UnsafeRawBufferPointer {
  public func withMemoryRebound<T, Result>(
    to type: T.Type,
    _ body: (_ buffer: UnsafeBufferPointer<T>) throws -> Result
  ) rethrows -> Result
  
  public func assumingMemoryBound<T>(to type: T.Type) -> UnsafeBufferPointer<T>
}

extension UnsafeMutableRawBufferPointer {
  public func withMemoryRebound<T, Result>(
    to type: T.Type,
    _ body: (_ buffer: UnsafeMutableBufferPointer<T>) throws -> Result
  ) rethrows -> Result

  public func assumingMemoryBound<T>(to type: T.Type) -> UnsafeMutableBufferPointer<T>
}
```


## Detailed design

Note: please see the [draft PR][draft-pr] to visualize the proposed changes rather than the proposed final state.

```swift
extension UnsafePointer {
  /// Executes the given closure while temporarily binding memory to
  /// the specified number of instances of type `T`.
  ///
  /// Use this method when you have a pointer to memory bound to one type and
  /// you need to access that memory as instances of another type. Accessing
  /// memory as a type `T` requires that the memory be bound to that type. A
  /// memory location may only be bound to one type at a time, so accessing
  /// the same memory as an unrelated type without first rebinding the memory
  /// is undefined.
  ///
  /// The region of memory that starts at this pointer and covers `count`
  /// strides of `T` instances must be bound to `Pointee`.
  /// Any instance of `T` within the re-bound region may be initialized or
  /// uninitialized. Every instance of `Pointee` overlapping with a given
  /// instance of `T` should have the same initialization state (i.e.
  /// initialized or uninitialized.) Accessing a `T` whose underlying
  /// `Pointee` storage is in a mixed initialization state shall be
  /// undefined behaviour.
  ///
  /// The following example temporarily rebinds the memory of a `UInt64`
  /// pointer to `Int64`, then accesses a property on the signed integer.
  ///
  ///     let uint64Pointer: UnsafePointer<UInt64> = fetchValue()
  ///     let isNegative = uint64Pointer.withMemoryRebound(to: Int64.self,
  ///                                                      capacity: 1) {
  ///         return $0.pointee < 0
  ///     }
  ///
  /// Because this pointer's memory is no longer bound to its `Pointee` type
  /// while the `body` closure executes, do not access memory using the
  /// original pointer from within `body`. Instead, use the `body` closure's
  /// pointer argument to access the values in memory as instances of type
  /// `T`.
  ///
  /// After executing `body`, this method rebinds memory back to the original
  /// `Pointee` type.
  ///
  /// - Note: Only use this method to rebind the pointer's memory to a type
  ///   that is layout compatible with the `Pointee` type. The stride of the
  ///   temporary type (`T`) may be an integer multiple or a whole fraction
  ///   of `Pointee`'s stride, for example to point to one element of
  ///   an aggregate.
  ///   To bind a region of memory to a type that does not match these
  ///   requirements, convert the pointer to a raw pointer and use the
  ///   `bindMemory(to:)` method.
  ///   If `T` and `Pointee` have different alignments, this pointer
  ///   must be aligned with the larger of the two alignments.
  ///
  /// - Parameters:
  ///   - type: The type to temporarily bind the memory referenced by this
  ///     pointer. The type `T` must be layout compatible
  ///     with the pointer's `Pointee` type.
  ///   - count: The number of instances of `T` in the re-bound region.
  ///   - body: A closure that takes a typed pointer to the
  ///     same memory as this pointer, only bound to type `T`. The closure's
  ///     pointer argument is valid only for the duration of the closure's
  ///     execution. If `body` has a return value, that value is also used as
  ///     the return value for the `withMemoryRebound(to:capacity:_:)` method.
  ///   - pointer: The pointer temporarily bound to `T`.
  /// - Returns: The return value, if any, of the `body` closure parameter.
  @_alwaysEmitIntoClient
  public func withMemoryRebound<T, Result>(
    to type: T.Type, capacity count: Int,
    _ body: (_ pointer: UnsafePointer<T>) throws -> Result
  ) rethrows -> Result
}
```

```swift
extension UnsafeMutablePointer {
  /// Executes the given closure while temporarily binding memory to
  /// the specified number of instances of the given type.
  ///
  /// Use this method when you have a pointer to memory bound to one type and
  /// you need to access that memory as instances of another type. Accessing
  /// memory as a type `T` requires that the memory be bound to that type. A
  /// memory location may only be bound to one type at a time, so accessing
  /// the same memory as an unrelated type without first rebinding the memory
  /// is undefined.
  ///
  /// The region of memory that starts at this pointer and covers `count`
  /// strides of `T` instances must be bound to `Pointee`.
  /// Any instance of `T` within the re-bound region may be initialized or
  /// uninitialized. Every instance of `Pointee` overlapping with a given
  /// instance of `T` should have the same initialization state (i.e.
  /// initialized or uninitialized.) Accessing a `T` whose underlying
  /// `Pointee` storage is in a mixed initialization state shall be
  /// undefined behaviour.
  ///
  /// The following example temporarily rebinds the memory of a `UInt64`
  /// pointer to `Int64`, then modifies the signed integer.
  ///
  ///     let uint64Pointer: UnsafeMutablePointer<UInt64> = fetchValue()
  ///     uint64Pointer.withMemoryRebound(to: Int64.self, capacity: 1) { ptr in
  ///         ptr.pointee.negate()
  ///     }
  ///
  /// Because this pointer's memory is no longer bound to its `Pointee` type
  /// while the `body` closure executes, do not access memory using the
  /// original pointer from within `body`. Instead, use the `body` closure's
  /// pointer argument to access the values in memory as instances of type
  /// `T`.
  ///
  /// After executing `body`, this method rebinds memory back to the original
  /// `Pointee` type.
  ///
  /// - Note: Only use this method to rebind the pointer's memory to a type
  ///   that is layout compatible with the `Pointee` type. The stride of the
  ///   temporary type (`T`) may be an integer multiple or a whole fraction
  ///   of `Pointee`'s stride, for example to point to one element of
  ///   an aggregate.
  ///   To bind a region of memory to a type that does not match these
  ///   requirements, convert the pointer to a raw pointer and use the
  ///   `bindMemory(to:)` method.
  ///   If `T` and `Pointee` have different alignments, this pointer
  ///   must be aligned with the larger of the two alignments.
  ///
  /// - Parameters:
  ///   - type: The type to temporarily bind the memory referenced by this
  ///     pointer. The type `T` must be layout compatible
  ///     with the pointer's `Pointee` type.
  ///   - count: The number of instances of `T` in the re-bound region.
  ///   - body: A closure that takes a mutable typed pointer to the
  ///     same memory as this pointer, only bound to type `T`. The closure's
  ///     pointer argument is valid only for the duration of the closure's
  ///     execution. If `body` has a return value, that value is also used as
  ///     the return value for the `withMemoryRebound(to:capacity:_:)` method.
  ///   - pointer: The pointer temporarily bound to `T`.
  /// - Returns: The return value, if any, of the `body` closure parameter.
  @_alwaysEmitIntoClient
  public func withMemoryRebound<T, Result>(
    to type: T.Type, capacity count: Int,
    _ body: (_ pointer: UnsafeMutablePointer<T>) throws -> Result
  ) rethrows -> Result
}
```

```swift
extension UnsafeBufferPointer {
  /// Executes the given closure while temporarily binding the memory referenced 
  /// by this buffer to the given type.
  ///
  /// Use this method when you have a buffer of memory bound to one type and
  /// you need to access that memory as a buffer of another type. Accessing
  /// memory as type `T` requires that the memory be bound to that type. A
  /// memory location may only be bound to one type at a time, so accessing
  /// the same memory as an unrelated type without first rebinding the memory
  /// is undefined.
  ///
  /// The number of instances of `T` referenced by the rebound buffer may be
  /// different than the number of instances of `Element` referenced by the
  /// original buffer. The number of instances of `T` will be calculated
  /// at runtime.
  /// 
  /// Any instance of `T` within the re-bound region may be initialized or
  /// uninitialized. Every instance of `Pointee` overlapping with a given
  /// instance of `T` should have the same initialization state (i.e.
  /// initialized or uninitialized.) Accessing a `T` whose underlying
  /// `Pointee` storage is in a mixed initialization state shall be
  /// undefined behaviour.
  ///
  /// Because this buffer's memory is no longer bound to its `Element` type
  /// while the `body` closure executes, do not access memory using the
  /// original buffer from within `body`. Instead, use the `body` closure's
  /// buffer argument to access the values in memory as instances of type
  /// `T`.
  ///
  /// After executing `body`, this method rebinds memory back to the original
  /// `Element` type.
  ///
  /// - Note: Only use this method to rebind the buffer's memory to a type
  ///   that is layout compatible with the currently bound `Element` type.
  ///   The stride of the temporary type (`T`) may be an integer multiple
  ///   or a whole fraction of `Element`'s stride.
  ///   To bind a region of memory to a type that does not match these
  ///   requirements, convert the buffer to a raw buffer and use the
  ///   `bindMemory(to:)` method.
  ///   If `T` and `Element` have different alignments, this buffer's
  ///   `baseAddress` must be aligned with the larger of the two alignments.
  ///
  /// - Parameters:
  ///   - type: The type to temporarily bind the memory referenced by this
  ///     buffer. The type `T` must be layout compatible
  ///     with the pointer's `Element` type.
  ///   - body: A closure that takes a  typed buffer to the
  ///     same memory as this buffer, only bound to type `T`. The buffer
  ///     parameter contains a number of complete instances of `T` based
  ///     on the capacity of the original buffer and the stride of `Element`.
  ///     The closure's buffer argument is valid only for the duration of the
  ///     closure's execution. If `body` has a return value, that value
  ///     is also used as the return value for the `withMemoryRebound(to:_:)`
  ///     method.
  ///   - buffer: The buffer temporarily bound to `T`.
  /// - Returns: The return value, if any, of the `body` closure parameter.
  @_alwaysEmitIntoClient
  public func withMemoryRebound<T, Result>(
    to type: T.Type,
    _ body: (_ buffer: UnsafeBufferPointer<T>) throws -> Result
  ) rethrows -> Result
}
```

```swift
extension UnsafeMutableBufferPointer {
  /// Executes the given closure while temporarily binding the memory referenced 
  /// by this buffer to the given type.
  ///
  /// Use this method when you have a buffer of memory bound to one type and
  /// you need to access that memory as a buffer of another type. Accessing
  /// memory as type `T` requires that the memory be bound to that type. A
  /// memory location may only be bound to one type at a time, so accessing
  /// the same memory as an unrelated type without first rebinding the memory
  /// is undefined.
  ///
  /// The number of instances of `T` referenced by the rebound buffer may be
  /// different than the number of instances of `Element` referenced by the
  /// original buffer. The number of instances of `T` will be calculated
  /// at runtime.
  /// 
  /// Any instance of `T` within the re-bound region may be initialized or
  /// uninitialized. Every instance of `Pointee` overlapping with a given
  /// instance of `T` should have the same initialization state (i.e.
  /// initialized or uninitialized.) Accessing a `T` whose underlying
  /// `Pointee` storage is in a mixed initialization state shall be
  /// undefined behaviour.
  ///
  /// Because this buffer's memory is no longer bound to its `Element` type
  /// while the `body` closure executes, do not access memory using the
  /// original buffer from within `body`. Instead, use the `body` closure's
  /// buffer argument to access the values in memory as instances of type
  /// `T`.
  ///
  /// After executing `body`, this method rebinds memory back to the original
  /// `Element` type.
  ///
  /// - Note: Only use this method to rebind the buffer's memory to a type
  ///   that is layout compatible with the currently bound `Element` type.
  ///   The stride of the temporary type (`T`) may be an integer multiple
  ///   or a whole fraction of `Element`'s stride.
  ///   To bind a region of memory to a type that does not match these
  ///   requirements, convert the buffer to a raw buffer and use the
  ///   `bindMemory(to:)` method.
  ///   If `T` and `Element` have different alignments, this buffer's
  ///   `baseAddress` must be aligned with the larger of the two alignments.
  ///
  /// - Parameters:
  ///   - type: The type to temporarily bind the memory referenced by this
  ///     buffer. The type `T` must be layout compatible
  ///     with the pointer's `Element` type.
  ///   - body: A closure that takes a mutable typed buffer to the
  ///     same memory as this buffer, only bound to type `T`. The buffer
  ///     parameter contains a number of complete instances of `T` based
  ///     on the capacity of the original buffer and the stride of `Element`.
  ///     The closure's buffer argument is valid only for the duration of the
  ///     closure's execution. If `body` has a return value, that value
  ///     is also used as the return value for the `withMemoryRebound(to:_:)`
  ///     method.
  ///   - buffer: The buffer temporarily bound to `T`.
  /// - Returns: The return value, if any, of the `body` closure parameter.
  @_alwaysEmitIntoClient
  public func withMemoryRebound<T, Result>(
    to type: T.Type,
    _ body: (_ buffer: UnsafeMutableBufferPointer<T>) throws -> Result
  ) rethrows -> Result
}
```

```swift
extension UnsafeRawPointer {
  /// Executes the given closure while temporarily binding memory to
  /// the specified number of instances of type `T`.
  ///
  /// Use this method when you have a pointer to raw memory and you need
  /// to access that memory as instances of a given type `T`. Accessing
  /// memory as a type `T` requires that the memory be bound to that type. A
  /// memory location may only be bound to one type at a time, so accessing
  /// the same memory as an unrelated type without first rebinding the memory
  /// is undefined.
  ///
  /// Any instance of `T` within the re-bound region may be initialized or
  /// uninitialized. The memory underlying any individual instance of `T`
  /// must have the same initialization state (i.e.  initialized or
  /// uninitialized.) Accessing a `T` whose underlying memory
  /// is in a mixed initialization state shall be undefined behaviour.
  ///
  /// The following example temporarily rebinds a raw memory pointer
  /// to `Int64`, then accesses a property on the signed integer.
  ///
  ///     let pointer: UnsafeRawPointer = fetchValue()
  ///     let isNegative = pointer.withMemoryRebound(to: Int64.self,
  ///                                                capacity: 1) {
  ///         return $0.pointee < 0
  ///     }
  ///
  /// After executing `body`, this method rebinds memory back to its original
  /// binding state. This can be unbound memory, or bound to a different type.
  ///
  /// - Note: The region of memory starting at this pointer must match the
  ///   alignment of `T` (as reported by `MemoryLayout<T>.alignment`).
  ///   That is, `Int(bitPattern: self) % MemoryLayout<T>.alignment`
  ///   must equal zero.
  ///
  /// - Note: The region of memory starting at this pointer may have been
  ///   bound to a type. If that is the case, then `T` must be
  ///   layout compatible with the type to which the memory has been bound.
  ///   This requirement does not apply if the region of memory
  ///   has not been bound to any type.
  ///
  /// - Parameters:
  ///   - type: The type to temporarily bind the memory referenced by this
  ///     pointer. This pointer must be a multiple of this type's alignment.
  ///   - count: The number of instances of `T` in the re-bound region.
  ///   - body: A closure that takes a typed pointer to the
  ///     same memory as this pointer, only bound to type `T`. The closure's
  ///     pointer argument is valid only for the duration of the closure's
  ///     execution. If `body` has a return value, that value is also used as
  ///     the return value for the `withMemoryRebound(to:capacity:_:)` method.
  ///   - pointer: The pointer temporarily bound to `T`.
  /// - Returns: The return value, if any, of the `body` closure parameter.
  public func withMemoryRebound<T, Result>(
    to type: T.Type,
    capacity count: Int,
    _ body: (_ pointer: UnsafePointer<T>) throws -> Result
  ) rethrows -> Result
}
```

```swift
extension UnsafeMutableRawPointer {
  /// Executes the given closure while temporarily binding memory to
  /// the specified number of instances of type `T`.
  ///
  /// Use this method when you have a pointer to raw memory and you need
  /// to access that memory as instances of a given type `T`. Accessing
  /// memory as a type `T` requires that the memory be bound to that type. A
  /// memory location may only be bound to one type at a time, so accessing
  /// the same memory as an unrelated type without first rebinding the memory
  /// is undefined.
  ///
  /// Any instance of `T` within the re-bound region may be initialized or
  /// uninitialized. The memory underlying any individual instance of `T`
  /// must have the same initialization state (i.e.  initialized or
  /// uninitialized.) Accessing a `T` whose underlying memory
  /// is in a mixed initialization state shall be undefined behaviour.
  ///
  /// The following example temporarily rebinds a raw memory pointer
  /// to `Int64`, then modifies the signed integer.
  ///
  ///     let pointer: UnsafeMutableRawPointer = fetchValue()
  ///     pointer.withMemoryRebound(to: Int64.self, capacity: 1) {
  ///         ptr.pointee.negate()
  ///     }
  ///
  /// After executing `body`, this method rebinds memory back to its original
  /// binding state. This can be unbound memory, or bound to a different type.
  ///
  /// - Note: The region of memory starting at this pointer must match the
  ///   alignment of `T` (as reported by `MemoryLayout<T>.alignment`).
  ///   That is, `Int(bitPattern: self) % MemoryLayout<T>.alignment`
  ///   must equal zero.
  ///
  /// - Note: The region of memory starting at this pointer may have been
  ///   bound to a type. If that is the case, then `T` must be
  ///   layout compatible with the type to which the memory has been bound.
  ///   This requirement does not apply if the region of memory
  ///   has not been bound to any type.
  ///
  /// - Parameters:
  ///   - type: The type to temporarily bind the memory referenced by this
  ///     pointer. This pointer must be a multiple of this type's alignment.
  ///   - count: The number of instances of `T` in the re-bound region.
  ///   - body: A closure that takes a typed pointer to the
  ///     same memory as this pointer, only bound to type `T`. The closure's
  ///     pointer argument is valid only for the duration of the closure's
  ///     execution. If `body` has a return value, that value is also used as
  ///     the return value for the `withMemoryRebound(to:capacity:_:)` method.
  ///   - pointer: The pointer temporarily bound to `T`.
  /// - Returns: The return value, if any, of the `body` closure parameter.
  public func withMemoryRebound<T, Result>(
    to type: T.Type,
    capacity count: Int,
    _ body: (_ pointer: UnsafeMutablePointer<T>) throws -> Result
  ) rethrows -> Result
}
```

```swift
extension UnsafeRawBufferPointer {
  /// Executes the given closure while temporarily binding the buffer to
  /// instances of type `T`.
  ///
  /// Use this method when you have a buffer to raw memory and you need
  /// to access that memory as instances of a given type `T`. Accessing
  /// memory as a type `T` requires that the memory be bound to that type.
  /// A memory location may only be bound to one type at a time, so accessing
  /// the same memory as an unrelated type without first rebinding the memory
  /// is undefined.
  ///
  /// Any instance of `T` within the re-bound region may be initialized or
  /// uninitialized. The memory underlying any individual instance of `T`
  /// must have the same initialization state (i.e.  initialized or
  /// uninitialized.) Accessing a `T` whose underlying memory
  /// is in a mixed initialization state shall be undefined behaviour.
  ///
  /// If the byte count of the original buffer is not a multiple of
  /// the stride of `T`, then the re-bound buffer is shorter
  /// than the original buffer.
  ///
  /// After executing `body`, this method rebinds memory back to its original
  /// binding state. This can be unbound memory, or bound to a different type.
  ///
  /// - Note: The buffer's base address must match the
  ///   alignment of `T` (as reported by `MemoryLayout<T>.alignment`).
  ///   That is, `Int(bitPattern: self.baseAddress) % MemoryLayout<T>.alignment`
  ///   must equal zero.
  ///
  /// - Note: A raw buffer may represent memory that has been bound to a type.
  ///   If that is the case, then `T` must be layout compatible with the
  ///   type to which the memory has been bound. This requirement does not
  ///   apply if the raw buffer represents memory that has not been bound
  ///   to any type.
  ///
  /// - Parameters:
  ///   - type: The type to temporarily bind the memory referenced by this
  ///     pointer. This pointer must be a multiple of this type's alignment.
  ///   - body: A closure that takes a typed pointer to the
  ///     same memory as this pointer, only bound to type `T`. The closure's
  ///     pointer argument is valid only for the duration of the closure's
  ///     execution. If `body` has a return value, that value is also used as
  ///     the return value for the `withMemoryRebound(to:capacity:_:)` method.
  ///   - buffer: The buffer temporarily bound to instances of `T`.
  /// - Returns: The return value, if any, of the `body` closure parameter.
  public func withMemoryRebound<T, Result>(
    to type: T.Type,
    _ body: (_ buffer: UnsafeBufferPointer<T>) throws -> Result
  ) rethrows -> Result

  /// Returns a typed buffer to the memory referenced by this buffer,
  /// assuming that the memory is already bound to the specified type.
  ///
  /// Use this method when you have a raw buffer to memory that has already
  /// been bound to the specified type. The memory starting at this pointer
  /// must be bound to the type `T`. Accessing memory through the returned
  /// pointer is undefined if the memory has not been bound to `T`. To bind
  /// memory to `T`, use `bindMemory(to:capacity:)` instead of this method.
  ///
  /// - Note: The buffer's base address must match the
  ///   alignment of `T` (as reported by `MemoryLayout<T>.alignment`).
  ///   That is, `Int(bitPattern: self.baseAddress) % MemoryLayout<T>.alignment`
  ///   must equal zero.
  ///
  /// - Parameter to: The type `T` that the memory has already been bound to.
  /// - Returns: A typed pointer to the same memory as this raw pointer.
  public func assumingMemoryBound<T>(
    to: T.Type
  ) -> UnsafeBufferPointer<T>
}
```

```swift
extension UnsafeMutableRawBufferPointer {
  /// Executes the given closure while temporarily binding the buffer to
  /// instances of type `T`.
  ///
  /// Use this method when you have a buffer to raw memory and you need
  /// to access that memory as instances of a given type `T`. Accessing
  /// memory as a type `T` requires that the memory be bound to that type.
  /// A memory location may only be bound to one type at a time, so accessing
  /// the same memory as an unrelated type without first rebinding the memory
  /// is undefined.
  ///
  /// Any instance of `T` within the re-bound region may be initialized or
  /// uninitialized. The memory underlying any individual instance of `T`
  /// must have the same initialization state (i.e.  initialized or
  /// uninitialized.) Accessing a `T` whose underlying memory
  /// is in a mixed initialization state shall be undefined behaviour.
  ///
  /// If the byte count of the original buffer is not a multiple of
  /// the stride of `T`, then the re-bound buffer is shorter
  /// than the original buffer.
  ///
  /// After executing `body`, this method rebinds memory back to its original
  /// binding state. This can be unbound memory, or bound to a different type.
  ///
  /// - Note: The buffer's base address must match the
  ///   alignment of `T` (as reported by `MemoryLayout<T>.alignment`).
  ///   That is, `Int(bitPattern: self.baseAddress) % MemoryLayout<T>.alignment`
  ///   must equal zero.
  ///
  /// - Note: A raw buffer may represent memory that has been bound to a type.
  ///   If that is the case, then `T` must be layout compatible with the
  ///   type to which the memory has been bound. This requirement does not
  ///   apply if the raw buffer represents memory that has not been bound
  ///   to any type.
  ///
  /// - Parameters:
  ///   - type: The type to temporarily bind the memory referenced by this
  ///     pointer. This pointer must be a multiple of this type's alignment.
  ///   - body: A closure that takes a typed pointer to the
  ///     same memory as this pointer, only bound to type `T`. The closure's
  ///     pointer argument is valid only for the duration of the closure's
  ///     execution. If `body` has a return value, that value is also used as
  ///     the return value for the `withMemoryRebound(to:capacity:_:)` method.
  ///   - buffer: The buffer temporarily bound to instances of `T`.
  /// - Returns: The return value, if any, of the `body` closure parameter.
  public func withMemoryRebound<T, Result>(
    to type: T.Type,
    _ body: (_ buffer: UnsafeMutableBufferPointer<T>) throws -> Result
  ) rethrows -> Result

  /// Returns a typed buffer to the memory referenced by this buffer,
  /// assuming that the memory is already bound to the specified type.
  ///
  /// Use this method when you have a raw buffer to memory that has already
  /// been bound to the specified type. The memory starting at this pointer
  /// must be bound to the type `T`. Accessing memory through the returned
  /// pointer is undefined if the memory has not been bound to `T`. To bind
  /// memory to `T`, use `bindMemory(to:capacity:)` instead of this method.
  ///
  /// - Note: The buffer's base address must match the
  ///   alignment of `T` (as reported by `MemoryLayout<T>.alignment`).
  ///   That is, `Int(bitPattern: self.baseAddress) % MemoryLayout<T>.alignment`
  ///   must equal zero.
  ///
  /// - Parameter to: The type `T` that the memory has already been bound to.
  /// - Returns: A typed pointer to the same memory as this raw pointer.
  public func assumingMemoryBound<T>(
    to: T.Type
  ) -> UnsafeMutableBufferPointer<T>
}
```

## Source compatibility

This proposal is source-compatible.
Some changes are compatible with existing correct uses of the API,
while others are additive.


## Effect on ABI stability

This proposal consists of ABI-preserving changes and ABI-additive changes.


## Effect on API resilience

The behaviour change for the `withMemoryRebound` is compatible with previous uses,
since restrictions were lifted.
Code that depends on the new semantics may not be compatible with old versions of these functions.
Back-deployment of new binaries will be supported by making the updated versions `@_alwaysEmitIntoClient`.
Compatibility of old binaries with a new standard library will be supported by ensuring that a compatible entry point remains.


## Alternatives considered

One alternative is to implement none of this change, and leave `withMemoryRebound` as is.
The usability problems of `withMemoryRebound` would remain.

Another alternative is to leave the type layout restrictions as they are for the typed `Pointer` and `BufferPointer` types,
but add the `withMemoryRebound` functions to the `RawPointer` and `RawBufferPointer` variants.
In that case, the stride restriction would be no more than a speedbump,
because it would be straightforward to bypass it by transiting through the appropriate `Raw` variant.
