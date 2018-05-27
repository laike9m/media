While I was learning move semantics, I always had a feeling, that even though I've known the concept quite well, I cannot fit it into the big picture of C++. Move semantics is not like some syntactic sugar that solely exists for convenience, it deeply affected the way people think and write C++ and has become one of the most important C++ idioms. But hey, the pond of C++ was already full of other idioms, when you throw move semantics in, mutual extrusion comes with it. Did move semantics break, enhance or replace other idioms? I don't know, but I want to find out.

## Value Semantics

Value semantics is what makes me start to think about this problem. Since there aren't many things in C++ with the name "semantics", I naturally thought, "maybe value and move semantics have some connections?". Turns out, it's not just connections, it's the origin:

> ***Move semantics is not an attempt to supplant copy semantics, nor undermine it in any way. Rather this proposal seeks to augment copy semantics.***
>
> \- [*Move Semantics Proposal*](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2002/n1377.htm#Copy%20vs%20Move), September 10, 2002

Perhaps you've noticed it uses the wording "copy semantics", in fact, "value semantics" and "copy semantics" are the same thing, and I'll use them interchangeably.

OK, so what is value semantics? isocpp has a [whole page](https://isocpp.org/wiki/faq/value-vs-ref-semantics) talking about it, but basically, value semantics means **assignment copies the value**, like `T b = a; `. That's the definition, but **often times value semantics just means to create, use, store the object itself, pass, return by value, rather than pointers or references**.

The opposite concept is reference semantics, where assignment copies the pointer. In reference semantics, what's important is identity, for example `T& b = a; ` , we have to remember that `b` is an alias of `a`, not anything else. But in value semantics, we don't care about identity at all, we only care about the value an object<sup>1</sup> holds. This is brought by the nature of copy, because a copy is ensured to give us two independent objects that hold the same value, you can't tell which one is the source, nor it has any affect on usage.

Unlike other languages(Java, C#, JavaScript), C++ is built on value semantics. By default, assignment does bit-wise-copy(if no user-defined copy ctor is involved), arguments and return values are copy-constructed(Yes I know there's RVO). Keeping value semantics is considered a good thing in C++. On the one hand, it's safer, because you don't need to worry about dangling pointers and all the creepy stuff; on the other hand, it's faster, because you have less indirection, see [here](https://isocpp.org/wiki/faq/value-vs-ref-semantics#compos-vs-heap) for the official explanation.

## Move Semantics: V8 Engine on the Value Semantics Car

Move semantics is not an attempt to supplant copy semantics. They are totally compatible with each other. I came up with this metaphor which I think describes their relation really well.

Imagine you have a car, it ran smoothly with the built-in engine. One day, you installed an extra V8 engine onto this car. Whenever you have enough fuel, the V8 engine is able to speed up your car, and that makes you happy.

So, the car is value semantics, and the V8 engine is move semantics. Installing an engine on your car does not require a new car, it's still the same car, just like using move semantics won't make you drop value semantics, because you're still operating on the object itself not its references or pointers. Further more, the ***move if you can, else copy*** strategy, implemented by the [binding preferences](https://www.codesynthesis.com/~boris/blog/2012/07/24/const-rvalue-references/), is exactly like way engine is chosen, that is to use V8 if you can(enough fuel), otherwise fall back to the original engine.

Now we have a pretty good understanding of Howard Hinnant(main author of the move proposal)'s [answer](https://stackoverflow.com/a/50200225/2142577) on SO:

> *Move semantics allows us to keep value semantics, but at the same time gain the performance of reference semantics in those cases where the value of the original (copied-from) object is unimportant to program logic.*

## Move Semantics and Performance Optimization

> ***Move semantics is mostly about performance optimization: the ability to move an expensive object from one address in memory to another, while pilfering resources of the source in order to construct the target with minimum expense.***
>
> \- [*Move Semantics Proposal*](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2002/n1377.htm#Copy%20vs%20Move)

As stated in the proposal, the main benefit people get from move semantics are performance boost. I'll give two examples here.

### The optimization you can see

Suppose we have a handler(whatever that is) which is expensive to construct. Now we want to store it into a map for future use.

```c++
std::unordered_map<string, Handler> handlers;
void RegisterHandler(const string& name, Handler handler) {
  handlers[name] = std::move(handler);
}
RegisterHandler("handler-A", build_handler());
```

This is a typical use of move, and of course it assumes `Handler` has a move ctor. By moving(not copying)-constructing a map value, a lot of time may be saved.

### The optimization you can't see

Howard Hinnant once mentioned in his [talk](https://youtu.be/vLinb2fgkHk?t=3m15s) that, the idea of move semantics came from optimizing `std::vector`. How?

A `std::vector<T>` object is basically a set of pointers to an internal data buffer on heap, like `begin()` and `end()`. Copying a vector is expensive due to allocating new memory for the data buffer. When move is used instead of copy, only the pointers get copied and point to the old buffer.

What's more, move also boosts vector `insert` operation. This is explained in the [vector Example](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2002/n1377.htm#vector%20Example) section in the proposal. Say we have a `std::vector<string>` with two elements `"AAAAA"` and `"BBBBB"`, now we want to insert `"CCCCC"` into at index 1. Assume the vector has enough capacity, the following graph demonstrates the process of inserting with copy vs move.

![](https://oftdwlsct.qnssl.com/move-semantics_vector.png)

Everything shown on the graph is on heap, including the vector's data buffer and each element string's data buffer. With copy, `str_b`'s data buffer has to be copied, which involves a buffer allocation then deallocation. With move, old `str_b`'s data buffer is reused by the `str_b` in the new address, no buffer allocation or deallocation is needed. This brings a huge performance boost, yet it means more than that, because now you can store expensive objects into a vector without sacrificing performance, while previously having to store pointers. This also helps extend usage of value semantics.

## Move Semantics and Resource Management

In the famous article *[Rule of Zero](https://blog.rmf.io/cxx11/rule-of-zero)*, the author wrote:

> *Using value semantics is essential for RAII, because references don’t affect the lifetime of their referrents.*

I found it to be a good starting point to discuss the correlation between move semantics and resource management.

As you may or may not know, RAII has another name called *Scope-Bound Resource Management* (SBRM), after the basic use case where the lifetime of an RAII object ends due to scope exit. Remember one advantage of using value semantics? Safety. We know exactly when an object's lifetime starts and ends, just by looking at the [storage duration](http://en.cppreference.com/w/cpp/language/storage_duration), and 99% of the time we'll find it at block scope, which makes it very simple. Things get a lot more complicated for pointers and references, now we have to worry about whether the object that is referenced or pointed to has been released. This is hard, what makes it worse is that these objects usually exist in different scope from its pointer or reference.

It's obvious why value semantics gets along well with RAII —— RAII binds the life cycle of a resource to the lifetime of an object, and with value semantics, you have a clear idea of objects' lifetime.

## But, resource is about identity…

Though value semantics and RAII seems to be a perfect match, in reality it's not. Why? Fundamentally speaking, because resource is about identity, while value semantics only cares about value. You have an open socket, you use the very socket; you have an open file, you use the very file. In the context of resource management, there aren't things with the same value. A resource represents itself, with unique identity.

See the contradiction here? If we stick with value semantics, it's hard to work with resources cause they cannot be copied. Prior to C++11, people solve this problem (mainly) in three ways:

* Use raw pointers;
* Write your own movable but non-copyable class(often Involves operation like `swap`, `splice` and private copy ctor);
* Use `auto_ptr`.

These solutions intended to solve the problem of unique ownership and ownership transferring, but they all have drawbacks. I won't talk about it here cause it's everywhere on the Internet. What I would like to address is that, even without move semantics, resource ownership management can be done, it's just that it takes more code and is often error-prone.

> *What is lacking is uniform syntax and semantics to enable generic code to move arbitrary objects (just as generic code today can copy arbitrary objects).*
>
> \- [*Move Semantics Proposal*](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2002/n1377.htm#Motivation)

Compared to statement in proposal, I like this [answer](https://stackoverflow.com/a/16823728/2142577) more:

> *In addition to the obvious efficiency benefit, this also affords a programmer a standards-compliant way to have objects that are **movable but not copyable**. Objects that are movable and not copyable convey a very clear boundary of resource ownership via standard language semantics …my point is that **move semantics is now a standard way to concisely express (among other things) movable-but-not-copyable objects.***

The above quote has done a pretty good job explaining what move semantics means to resource ownership management in C++. Resource should naturally be movable(by "movable" I mean transferrable) but not copyable, now with the help of move semantics(well actually a whole lot of change at language level to support it), there's a standard way to do this right and efficiently.

## The Rebirth of Value Semantics

Finally, we are able to talk about the other aspect(besides performance) of augmentation that move semantics brought to value semantics.

Stepping through the above discussion, we've seen why value semantics fits the RAII model, but at the same time not compatible with resource management. With the arise of move semantics, the necessary materials to fill this gap is finally prepared. So here we have, smart pointers!

Needless to say the importance of `[std::unique_ptr](http://en.cppreference.com/w/cpp/memory/unique_ptr)` and `[std::shared_ptr](http://en.cppreference.com/w/cpp/memory/shared_ptr)`, here I'd like to emphasize three things:

* They follow RAII;
* They take huge advantage of move semantics(especially for unique_ptr);
* They help keep value semantics.

For the third point, if you've read *[Rule of Zero](https://blog.rmf.io/cxx11/rule-of-zero)*, you know what I'm talking about. No need to use raw pointers to manage resources, EVER, just use unique_ptr directly or store as member variable, and you're done. When transferring resource ownership, the implicitly constructed move ctor is able to do the job well. Better yet, the current specification ensures that, a named value in the return statement in the worst case(i.e. without elisions) is treated as an rvalue. It means, **returning by value should be the default choice for unique_ptr**.

```c++
std::unique_ptr<ExpensiveResource> foo() {
  auto data = std::make_unique<ExpensiveResource>();
  return data;
}
std::unique_ptr<ExpensiveResource> p = foo();  // a move at worst
```

See [here](https://stackoverflow.com/a/4316948/2142577) for a more detailed explanation. In fact, **when used as function parameters, unique_ptr should also be passed by value.** I'll probably write an article about it, if time is available.

Besides smart pointers, `std::string` and `std::vector ` are also RAII wrappers, and the resource they manage is heap memory. For these classes, return by value is still preferred. I'm not too sure about other things like `std::thread` or `std::lock_guard` cause I haven't got chance to use them.

To summarize, by utilizing smart pointers, value semantics now truly gains the compatibility with RAII. At its core, this is powered by move semantics.

## Summary

So far we've gone through a lot of concepts and you probably feel overwhelmed, but the points I want to convey is simple:

1. Move semantics boosts performance while keeping value semantics;
2. Move semantics helps bring every piece of resource management together to become what it is today. Especially, it is the key that makes value semantics and RAII truly work together, as it should have been long ago.

I'm a learner on this topic myself, so feel free to point out you feel is something is wrong, I really appreciate it.


\[1\]: Here object means "*a piece of memory that has an address, a type, and is capable of storing values*", from [Andrzej's C++ blog](https://akrzemi1.wordpress.com/2012/02/03/value-semantics/).