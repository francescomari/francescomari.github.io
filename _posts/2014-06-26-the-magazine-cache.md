---
layout: post
title:  "The Magazine cache"
date:   2014-06-26
---

Sooner or later everyone needs a cache in its application. There are many
caching strategies available to the wise developer: least recently used (LRU),
most recently used (MRU), least frequently used (LFU), and a lot more. In this
post I want to discuss about [Magazine](https://github.com/bigeasy/magazine), a
Node module implementing a flexible LRU cache.

Magazine has a very minimal but functional API but, before going through it, I
would like to introduce the concepts used by the module.

The data saved in Magazine can be stored and accessed by using a **magazine**
object. Once you have a magazine, you can store key-value pairs in it. Every key
stored in a magine is wrapped in a **cartridge**, which act as a slot for a
single piece of data.

You can have multiple magazines, and all of them will be grouped in a **cache**
object. The following example shows how to create a cache, create a magazine
from it, and store some values in the magazine.

```js
var numbers = new Cache();

var odds = numbers.createMagazine();
odds.hold('one', 1);
odds.hold('three', 3);

var evens = numbers.createMagazine();
evens.hold('two', 2);
evens.hold('four', 4);

assert(numbers.count === 4);
assert(odds.count === 2);
assert(evens.count === 2);
```

The cache is not just a passive container of magazines. In fact, it knows about
every key-value pair stored in any magazine. The previous example defines a
cache with two magazines, where each magazine holds two cartridges. Every
magazine is only aware of the cartridges it holds, but the cache is able to
reach every cartridge, indipendently from the magazine holding them.

A cartridge is more than a simple key wrapper. Associated to every cartridge
there is a reference counter which represents the number of holders of that
cartridge. When you initially save a key-value pair in a magazine, the
corresponding cartridge has an initial reference counter of `1`. Every time you
invoke the `hold()` method on the same key, the reference counter of the
corresponding cartridge is incremented.

There is no reccommended way to inspect the reference counter associated to a
cartridge, but you can decrement it by invoking the `release()` method on a
cartridge.

```js
var cache = new Cache();
var magazine = cache.createMagazine();
var cartridge = magazine.hold('one', 1);

cartridge.release();
```

The strategy is really simple. If you put a key-value pair in a magazine, you
automatically obtain a reference to it. When you don't need that key-value pair
anymore, just release the reference. A key-value pair is not automatically
removed from the magazine, even if its reference counter drops down to zero: you
have to explicitly remove it from the magazine. If you try to remove a cartridge
with a reference counter greater than zero, an error will be thrown.

```js
var cache = new Cache();
var magazine = cache.createMagazine();

magazine.hold('one', 1);
assert(magazine.count === 1);

magazine.get('one').release();
magazine.remove('one');
assert(magazine.count === 0);
```

There is one more functionality implemented by the Magazine module which is very
useful when you want to keep the size of the cache under control. Every
cartridge in the cache has a **heft** representing its weight. The idea is that
every value stored in the cache contributes to the overall weight of the cache.
The more values you put in the cache, and the higher their heft is, the more
your cache will be heavy.

You can adjust the heft of a cartridge at any time by using the `adjustHeft()`
method. Usually you do it when you `hold()` a key-value pair, because it is
likely that the heft is a function of the value assigned to the cartridge. Every
cartridge contributes to the heft of a magazine, and every magazine contributes
to the heft of the whole cache. You can inspect the `heft` property for
cartridges, magazines and caches to know the weight of these items.

The heft is an important information when purging the cache. The cache or a
magazine can be purged at any time to bring it under a certain size. The
implementation keeps removing cartridges until the heft of the cache or of the
magazine reeaches a desired amount. Only cartridges which are not referenced can
be removed. Any cartridge with a reference count grater than zero is always kept
in the cache, no matter how big its heft is.

The following example is a little bit long, but I hope that the inline comments
help clarifying what I am doing. Pay particular attention to the way the heft
changes for the magazines and the cache.

```js
var cache = new Cache();
var first = cache.createMagazine();
var second = cache.createMagazine();

// Populate the two magazines

first.hold('a', 1).adjustHeft(10);
second.hold('b', 2).adjustHeft(15);

assert(first.heft === 10);
assert(second.heft === 15);
assert(cache.heft === 25);

// If we hold references, we cannot remove cartridges

cache.purge(0);

assert(cache.heft === 25);

// Release a reference, purge the whole cache

first.get('a').release();
cache.purge(0);

assert(first.heft === 0);
assert(second.heft === 15);
assert(cache.heft === 15);

// Release another reference, purge the magazine

second.get('b').release();
second.purge(0);

assert(first.heft === 0);
assert(second.heft === 0);
assert(cache.heft === 0);
```

There are two important observations to make. First, a cache purge is just a
best-effort operation. By invoking `purge()` you express your desire to bring
the heft of the cache under a certain limit. If all the cartridges in the cache
are on hold because someone is referencing them, there is no way to remove them:
your cache keeps the same heft even if you invoked a purge operation. Second,
the cache is always updated even if you invoke `purge()` on a magazine. This
allows you to use the cache object as a central point of control, and to invoke
a global purge for every magazine used by the application.

The Magazine module is very well written and self contained. It is a good
example of how good modules should be developed and I reccommend you to go and
read the source code. It is a smooth experience which can help your brain stay
fit.
