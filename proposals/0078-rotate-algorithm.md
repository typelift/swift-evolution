# Implement a rotate algorithm, equivalent to std::rotate() in C++

* Proposal: [SE-0078](0078-rotate-algorithm.md)
* Authors: [Nate Cook](https://github.com/natecook1000), [Sergey Bolshedvorsky](https://github.com/bolshedvorsky)
* Status: **Active review May 3...9, 2016**
* Review manager: [Chris Lattner](http://github.com/lattner)

## Introduction

This proposal is to add rotation and in-place reversing methods to Swift's
standard library collections.

Swift-evolution thread: [link to the discussion thread for that proposal](https://lists.swift.org/pipermail/swift-evolution/Week-of-Mon-20151214/002213.html)

## Motivation

Rotation is one of the most important algorithms. It is a fundamental tool used in many
other algorithms with applications even in GUI programming.

The "rotate" algorithm performs a left rotation on a range of elements.
Specifically, it swaps the elements in the range `startIndex..<endIndex`
according to a `middle` index in such a way that the element at `middle` becomes
the first element of the new range and `middle - 1` becomes the last element.
The result of the algorithm is the new index of the element that was originally
first in the collection.

```swift
var a = [10, 20, 30, 40, 50, 60, 70]
let i = a.rotate(firstFrom: 2)
// a == [30, 40, 50, 60, 70, 10, 20]
// i == 5
```

The index returned from a rotation can be used as the `middle` argument in a second
rotation to return the collection to its original state, like this:

```swift
a.rotate(firstFrom: i)
// a == [10, 20, 30, 40, 50, 60, 70]
```

There are three different versions of the rotate algorithm, optimized for
collections with forward, bidirectional, and random access indices. The
complexity of the implementation of these algorithms makes the generic rotate
algorithm a perfect candidate for the standard library.

<details>
  <summary>**Example C++ Implementations**</summary>

**Forward indices** are the simplest and most general type of index and support
only one-directional traversal.

The C++ implementation of the rotate algorithm for the `ForwardIterator`
(`ForwardIndex` in Swift) may look like this:

```C++
template <ForwardIterator I>
I rotate(I f, I m, I l, std::forward_iterator_tag) {
    if (f == m) return l;
    if (m == l) return f;
    pair<I, I> p = swap_ranges(f, m, m, l);
    while (p.first != m || p.second != l) {
        if (p.second == l) {
            rotate_unguarded(p.first, m, l);
            return p.first;
        }
        f = m;
        m = p.second;
        p = swap_ranges(f, m, m, l);
    }
    return m;
}
```

**Bidirectional indices** are a refinement of forward indices that
additionally support reverse traversal.

The C++ implementation of the rotate algorithm for the `BidirectionalIterator`
(`BidirectionalIndex` in Swift) may look like this:

```C++
template <BidirectionalIterator I>
I rotate(I f, I m, I l, bidirectional_iterator_tag) {
    reverse(f, m);
    reverse(m, l);
    pair<I, I> p = reverse_until(f, m, l);
    reverse(p.first, p.second);
    if (m == p.first) return p.second;
    return p.first;
}
```

**Random access indices** access to any element in constant time (both far and fast).

The C++ implementation of the rotate algorithm for the `RandomAccessIterator`
(`RandomAccessIndex` in Swift) may look like this:

```C++
template <RandomAccessIterator I>
I rotate(I f, I m, I l, std::random_access_iterator_tag) {
    if (f == m) return l;
    if (m == l) return f;
    DifferenceType<I> cycles = gcd(m - f, l - m);
    rotate_transform<I> rotator(f, m, l);
    while (cycles-- > 0) rotate_cycle_from(f + cycles, rotator);
    return rotator.m1;
}
```

</details>


## Proposed solution

The Swift standard library should provide generic implementations of the
"rotate" algorithm for all three index types, in both mutating and nonmutating
forms. The mutating form is called `rotate(firstFrom:)` and rotates the elements
of a collection in-place. The nonmutating form of the "rotate"
algorithm is called `rotated(firstFrom:)` and return views onto the original
collection with the elements rotated, preserving the level of the original
collection's index type.

In addition, since the bidirectional algorithm depends on reversing the
collection's elements in-place, the standard library should also provide an
in-place `reverse()` method to complement the existing nonmutating `reversed()`
collection method.

## Detailed design

The mutating methods will have the following declarations:

```swift
extension MutableCollection {
    /// Rotates the elements of the collection so that the element
    /// at `middle` ends up first.
    ///
    /// - Returns: The new index of the element that was first
    ///   pre-rotation.
    @discardableResult
    public mutating func rotate(firstFrom middle: Index) -> Index
}

extension MutableCollection where Self: BidirectionalCollection,
  SubSequence: MutableCollection, SubSequence: BidirectionalCollection {
    /// Reverses the elements of the collection in-place.
    public mutating func reverse()

    /// Rotates the elements of the collection so that the element
    /// at `middle` ends up first.
    ///
    /// - Returns: The new index of the element that was first
    ///   pre-rotation.
    @discardableResult
    public mutating func rotate(firstFrom middle: Index) -> Index
}

extension MutableCollection where Self: RandomAccessCollection {
    /// Reverses the elements of the collection in-place.
    public mutating func reverse()

    /// Rotates the elements of the collection so that the element
    /// at `middle` ends up first.
    ///
    /// - Returns: The new index of the element that was first
    ///   pre-rotation.
    @discardableResult
    public mutating func rotate(firstFrom middle: Index) -> Index
}
```

The nonmutating methods will return a tuple containing both a rotated view of
the original collection and the new index of the element that was previously
first. For forward- and bidirectional-collections, these methods will return
`FlattenCollection` and `FlattenBidirectionalCollection` instances, respectively:

```swift
extension Collection where SubSequence: Collection {
    /// Returns a rotated view of the elements of the collection, where the
    /// element at `middle` ends up first, and the index of the element that
    /// was previously first.
    func rotated(firstFrom middle: Index) ->
        (collection: FlattenCollection<[Self.SubSequence]>,
        rotatedStart: FlattenCollectionIndex<[Self.SubSequence]>)
}

extension Collection where SubSequence: BidirectionalCollection {
    /// Returns a rotated view of the elements of the collection, where the
    /// element at `middle` ends up first, and the index of the element that
    /// was previously first.
    func rotated(firstFrom middle: Index) ->
        (collection: FlattenBidirectionalCollection<[Self.SubSequence]>,
        rotatedStart: FlattenBidirectionalCollectionIndex<[Self.SubSequence]>)
}
```

There isn't a random-access `FlattenCollection`, since it can't walk an unknown
number of subcollections in O(1) time. However, a rotated random-access
collection has exactly two subcollections, so a specialized type can be created
to provide the rotated elements with a random-access index. This type will be
added as `RotatedCollection`:

```swift
/// A rotated view of an underlying random-access collection.
public struct RotatedCollection<
    Base: RandomAccessCollection>: RandomAccessCollection {
    // standard collection innards
}

/// The index type for a `RotatedCollection`.
public struct RotatedCollectionIndex<Base: Comparable>: Comparable {
    // standard index innards
}

extension RandomAccessCollection {
    /// Returns a rotated view of the elements of the collection, where the
    /// element at `middle` ends up first, and the index of the element that
    /// was previously first.
    func rotated(firstFrom middle: Index) -> (collection: RotatedCollection<Self>,
        rotatedFirst: RotatedCollectionIndex<Self>)
}
```

Lazy collections will also be extended with rotate methods that provide lazy rotation:

```swift
extension LazyCollectionProtocol where Elements.SubSequence: Collection,
    Index == Elements.Index {
    /// Returns a rotated view of the elements of the collection, where the
    /// element at `middle` ends up first, and the index of the element that
    /// was previously first.
    public func rotated(firstFrom middle: Elements.Index) ->
        (collection: LazyCollection<FlattenCollection<[Elements.SubSequence]>>,
        rotatedStart: FlattenCollectionIndex<[Elements.SubSequence]>)
}

extension LazyCollectionProtocol where Self: BidirectionalCollection,
    Elements.SubSequence: BidirectionalCollection, Index == Elements.Index {
    /// Returns a rotated view of the elements of the collection, where the
    /// element at `middle` ends up first, and the index of the element that
    /// was previously first.
    public func rotated(firstFrom middle: Elements.Index) -> (collection:
        LazyCollection<FlattenBidirectionalCollection<[Elements.SubSequence]>>,
        rotatedStart: FlattenBidirectionalCollectionIndex<[Elements.SubSequence]>)
}

extension LazyCollectionProtocol where Self: RandomAccessCollection,
    Elements: RandomAccessCollection, Index == Elements.Index {
    /// Returns a rotated view of the elements of the collection, where the
    /// element at `middle` ends up first, and the index of the element that
    /// was previously first.
    public func rotated(firstFrom middle: Elements.Index) ->
        (collection: LazyCollection<RotatedCollection<Elements>>,
        rotatedStart: RotatedCollectionIndex<Elements>)
}
```

Rotation algorithms, structs and extensions will be implemented in
`stdlib/public/core/Rotate.swift`, with tests in `test/1_stdlib/Rotate.swift`.
The new in-place `reverse()` method will be added to
`stdlib/public/core/Reverse.swift`.

## Usage examples

*In-place rotation:*

```swift
var numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9]
numbers.rotate(firstFrom: 3)
expectEqual(numbers, [4, 5, 6, 7, 8, 9, 1, 2, 3])

var toMerge = [2, 4, 6, 8, 10, 3, 5, 7, 9]
let i = toMerge[2..<7].rotate(firstFrom: 5)
expectEqual(toMerge, [2, 4, 3, 5, 6, 8, 10, 7, 9])
expectEqual(i, 4)
```

*Nonmutating rotation:*

```swift
let numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9]
let (rotated, i) = numbers.rotated(firstFrom: 3)
expectEqual(rotated, [4, 5, 6, 7, 8, 9, 1, 2, 3])
```

*Lazy rotation:*

```swift
let numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9]
let (rotated, i) = numbers.lazy.rotated(firstFrom: 3)
expectEqual(rotated.first!, 4)
```

*Reversing in place:*

```swift
var numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9]
numbers.reverse()
expectEqual(numbers, [9, 8, 7, 6, 5, 4, 3, 2, 1])
numbers[0..<5].reverse()
expectEqual(numbers, [5, 6, 7, 8, 9, 4, 3, 2, 1])
```

## Impact on existing code

This is an additive feature that doesn’t impact existing code.

## Alternatives considered

The alternative is to keep the current behaviour, but the user will need to develop
their custom implementation of the rotate algorithms tailored for their needs.
