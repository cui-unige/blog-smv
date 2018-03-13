---
layout: post
title: "Let's stop overloading the assignment operator"
subtitle: "Featuring a single assignment operator no longer makes sense."
tags: [Post, Programming Languages]
feature-img: "code.jpg"
---

An [imperative programming languages](https://en.wikipedia.org/wiki/Imperative_programming)
(e.g. C, Java, Python, ...)
uses statements to change some state of the program.
In modern languages, these changes are often expressed in terms of
[variable assignment](https://en.wikipedia.org/wiki/Assignment_(computer_science)).
Take the following program example, written in C:
```c
int main() {
  int x = 0;
  x = 2;
  return x;
}
```
When the function starts,
a new [variable](https://en.wikipedia.org/wiki/Variable_(computer_science)) `x` is created,
and is _assigned_ to the value `0`.
Then, the following statement changes this assignment to make `x` assigned to the value `2` instead.
In both instances, we use an _assignment operator_ (in this case `=`)
to assign the value on its right to the variable on its left.

The concept of assignment sounds pretty straightforward.
You have an expression on the right (also referred to as a _r-value_)
that you want to assign to the expression on the left (also referred to as a _l-value_).
The l-value isn't necessary a variable name.
It can be an expression that evaluates to a container location (e.g. a pointer dereference).
For example:
```c
int main() {
  int* x = (int*)malloc(sizeof(int));
  *x = 2;
}
```
Here, the l-value is an expression that can read as "at the address of `x`",
so the whole statement reads as "assign the value `2` at the address of `x`".
Still pretty straightforward right?

Actually, things start to head south
when considering languages that add additional semantics to this operator.
As one may imagine, people grew tired of writing `*` and `&` symbols everywhere,
and so language designers decided to write compilers/interpreters smart enough to figure out
where to put them.
For instance, almost everything is a reference (i.e. a pointer) in Python,
except a bunch of primitive types such as `int` or `string`.
So if we write:
```python
b = [0]
b = b + [1]
```
a Python's interpreter is smart enough to understand
that these two lines translate to something like:
```c
// 1st line
int* b = (int*)malloc(sizeof(int) * 1);
b[0] = 0;

// 2nd line
int* tmp = (int*)malloc(sizeof(int) * 2);
memcpy(tmp, b, 1);
b = tmp;
b[1] = 1;
```
Nice! Now we don't need to worry about those pesky `*` anymore!
But unfortunately we lost a valuable information: whether or not `b` is actually a pointer.
And why does it matter?
Because now we can't know the syntax alone what happens when `b` is used as a function argument.
Consider the following program:
```python
def f(arg):
  arg = arg + arg

a = 0
f(a)
print(a)  # Prints "0"

b = [0]
f(b)
print(b)  # Prints "[0, 0]"
```
Wait what?!
Ah yes, remember that variables typed with Python's `int` aren't pointers,
because `int` is a primitive type.
But `list` isn't, so `b` is a pointer.
That means when we call `f` with `a`, we pass the latter [by value](https://en.wikipedia.org/wiki/Evaluation_strategy#Call_by_value),
while when we call `f` with `b`, we pass it [by reference](https://en.wikipedia.org/wiki/Evaluation_strategy#Call_by_reference).

Okay, that's bad. But as long as we remember which type is passed by value we're safe right?
After all, the list of primitive types usually isn't that long.
Well, then what about languages whose difference is **not** limited to a bunch of primitive types?
For instance, Swift features what it calls _value types_ and _reference types_,
and one can define as many has she/he wants.
This makes for a pretty hard to understand semantics, that many have already explored:
* https://developer.apple.com/swift/blog/?id=10
* https://medium.com/@andrea.prearo/reference-and-value-types-in-swift-dad40ea76226
* https://khawerkhaliq.com/blog/swift-value-types-reference-types/
* ...

And things get even worse as we keep overloading the assignment operator with more semantics.
A recent (well it's been 7 years, but many would still consider it recent) addition to C++ is the so-called
[move semantics](https://www.cprogramming.com/c++11/rvalue-references-and-move-semantics-in-c++11.html).
This is a pretty big optimization that can avoid unnecessary copies.
Consider the following example:
```c++
std::vector<int> f() {
  std::vector result;
  for (std::size_t i = 0, i < 1000000; ++i) {
    result.push_back(i);
  }
  return result
}

int main() {
  std::vector<int> v = f();
  return 0;
}
```
This code would make every C++03 programmer cringe,
but it is perfectly fine in 2018.
That's because back before move semantics was implemented,
the vector built within `f` would get copied and assigned to `v` (in `main`),
and destroyed right after,
hence needlessly copying an array of one million integers.
Without delving too much into the details,
move semantics simply allows to _transfer_ the memory from the return value of the function to `v`.
That's great isn't it?
But why is move semantics applied, rather than copying?
This is completely opaque, because the same operator (i.e. `=`)
has different meaning depending on the context, the types involved, etc.
Once again, this makes for a pretty hard to understand semantics,
which has also been a source of inspiration for many other blog posts:
* https://www.cprogramming.com/c++11/rvalue-references-and-move-semantics-in-c++11.html
* https://alexpolt.github.io/empty-value.html
* https://akrzemi1.wordpress.com/2011/08/11/move-constructor/
* ...

And this goes without even mentioning Rust [borrow checker hell](https://www.reddit.com/r/rust/comments/2r5t76/stuck_in_borrow_checker_hell_while_porting/)...

So what can we do about it?
Well, let's **stop overloading the assignment operator**.
This only brings confusion, and is very harmful in so many situations.
Source code should be unambiguous, not only to a compiler,
but also the human who writes and reads it.
Consider the following example, written in [Anzen](http://anzen-lang.org):
```anzen
struct Message {
  let value: @mut String
  mut fun __cpy__(other: Message) {
    self.value = other.value
  }
  fun __del__() {
    print(self.value)
  }
}

// ...

do {
  var msg1 <- Message(value <- "Hello ,")
  let msg2 <- @mut Message(value <- "Country")
  do {
    let msg3: @mut &- msg2
    msg3.value = "World"
    let msg4: @mut = msg2
    msg4.value = "!"
    msg1 <- msg4 // Prints "Hello ,"
  }
} // Prints "World" followed by "!"
```
Anzen has 3 assignment operators:
* a copy assignment operator `=` that always makes a copy of its r-value;
* a reference assignment operator `&-` that always makes a reference on its r-value; and
* a move assignment operator `<-` that always transfers the memory of its r-value.
No need to guess the semantics under which a variable assignment behaves,
just because of the type of its l/r-value, or the context it is used in.

Granted, Anzen's assignment semantics are complex, but so are those of C++, Rust, Swift, ...
At least, the kind of assignment performed at each line is always explicit from the syntax,
and I do believe that's the way it should be, in all languages.
