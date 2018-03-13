---
layout: post
title: "My love/hate relationship with Swift"
subtitle: "One of the reasons Swift is so great, and one of the reasons it really sucks."
tags: [Post, Programming Languages]
feature-img: "code.jpg"
---

I love [Swift](https://swift.org)!
It's a great modern programming language,
with performances [comparable to C/C++](https://benchmarksgame.alioth.debian.org/u64q/compare.php?lang=swift&lang2=gpp),
and a arguably intuitive syntax.
Just for a taste of the language,
here's an example:
```swift
enum List<Element>: Sequence, ExpressibleByArrayLiteral {
  indirect case node(element: Element, next: List)
  case empty

  init(arrayLiteral elements: Element...) {
    self = elements.reversed().reduce(.empty) { List.node(element: $1, next: $0) }
  }

  func makeIterator() -> AnyIterator<Element> {
    var node = self
    return AnyIterator {
      guard case let .node(element: e, next: nextNode) = node else { return nil }
      node = nextNode
      return e
    }
  }
}

let l: List = [1, 2, 3]
print(l.map { $0 * $0 }) // Prints "[1, 4, 9]"
```
Et voilà, in 17 lines I elegantly described a generic linked list type,
that supports all sequence operations such as `map`, `reduce`, `filter`, etc.
It even features a `sorted` method, if and only if it is specialized with a `Comparable` type.
And I got almost all of this for free.
The same code in C++ would probably be hundreds of lines long.

But I also have a huge problem with Swift,
and it's about the semantics of its assignments.
Let's say I create an [`Array`](https://developer.apple.com/documentation/swift/array)
with a couple of string values:
```swift
var a = ["Hello,", "World"]
print(a.joined(separator: " ")) // Prints "Hello, World"
```
Looks like I forgot the customary "!" at the end of the second word.
No problem:
```swift
a[1] += "!"
print(a.joined(separator: " ")) // Prints "Hello, World!"
```
So far so good.
But what if I assign `a[1]` to a value before mutating it?
```swift
var a = ["Hello,", "World"]
var world = a[1]
world += "!"
print(a.joined(separator: " ")) // Prints "Hello, World"
```
Looks like the "!" didn't make the cut.
The reason is that [`String`](https://developer.apple.com/documentation/swift/string) respect
Swift's [value](https://developer.apple.com/swift/blog/?id=10) semantics.
Values in Swift are represented by immutable data, allocated on the stack.
So when it executes a statement like `x += y`, Swift actually _reassigns_ the name `x`
to the result of `x + y`.
The first case has the expected result because what we reassigned was the 2nd position of the array `a`.
However, in the second case, the assignment `world = a[1]` dissociates
the actual memory location represented by `world` from that represented by `a[1]`.

Okay, fine. It may not be ideal not to have a way to alias a the position of an array,
but that's the semantics Swift chose.
The real problem is that this isn't the semantics it chose for **all** its values.
Consider this example:
```swift
class StringRef: StringProtocol {
  // A lot of protocol requirement shenanigans here ...
}
var a: StringRef = ["Hello,", "World"]
var world = a[1]
world += "!"
print(a.joined(separator: " ")) // Prints "Hello, World!"
```
Wait what?

The code of this new example looks practically identical to the former one,
yet the result is different.
The reason is that the type `StringRef` we created is a class,
and Swift's classes have a reference semantics.
So when we write `var world = a[1]`,
we no longer copy the value of `a[1]`,
but the pointer that is located at `a[1]`.
Therefore, the next statement do mutates the memory location represented by `a[1]`.
