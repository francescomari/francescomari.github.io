---
layout: post
title:  "Process lists lazily in JavaScript"
date:   2014-05-11
---

A common idiom in JavaScript is to apply multiple calls to `filter()` or `map()`
to an array to create a new output array with desired properties. While this is
a well known approach, this may not be the most efficient one. In the first
place, the input array must already be in memory. This also means that the array
must be known in advance: abstractions like infinite arrays or infinite
sequences are not possible out of the box. Here I present a simple abstraction,
inspired by pure, lazy, functional languages, to model a list and some high
level operations on it.

When handling lists, it is useful to approach to a recursive definition. A list
`[1, 2, 3]` can be considered like the union of the element `1` with the list
`[2, 3]`. Recusively, the list `[2, 3]` can be also considered like the union of
`2` with the list `[3]`. Finally, the list `[3]` can be considered like the
union of `3` with the empty list `[]`. I can model this recurisve behavior with
a simple pair of functions.

```js
function empty() {
  return [];
}

function cons(x, xs) {
  return [x].concat(xs);
}
```

These two functions, unfortunately, are not solving any of the problems that
JavaScript arrays have. They are creating an in-memory representation of the
list. The current defintions of `empty` and `cons` don't support the correct
level of abstraction that I want to provide. I should rewrite the functions as
follow.

```js
function empty() {
  return function () {
    return null;
  };
}

function cons(head, rest) {
  return function () {
    return {head: head, rest: rest};
  };
}
```

Here I provide a new level of indirection by saying that `empty` and `cons`
don't return a list, but another function. For `empty`, the returned function
simply returns `null`, meaning that no other element is contained in the list.
For `cons`, instead, the returned function returns an object with a `head`,
which is the current element, and a `rest`, which is another list containing
successive elements. Given these two functions, I can write the list `[1, 2, 3]`
using the following notation.

```js
var list = cons(1, cons(2, cons(3, empty())));
```

This definition also allows me to create an infinite list returning always the
same number. I can create a function which invokes itself and returns a never
ending list, where each element is always the same.

```js
function repeat(head) {
  return function () {
    return {head: head, rest: repeat(head)};
  };
}
```

Please note that there is no explicit recursion in this function. The function
would have been strictly recursive if `repeat` would call itself from within its
body. This is not the case, because `repeat` returns a function which,
eventually, will call `repeat` again. Calling this function is perfectly safe,
no stack overflows will be triggered.

Now that I have lists, in both finite and infinite variants, the first problem I
have to face is how to traverse them. I will create a function which, for each
element in the list, executes a user-provided function.

```js
function each(list, handler) {
  var current = list();

  while (current) {
    handler(current.head);
    current = current.rest();
  }
}
```

The code is not more complicated than a usual array traversal. A list is
represented by a function which either returns an element or a `null` value. If
a valid object is returned, I assume that the returned object has a `head`
containing the current element a `rest` property representing the remaining
elements in the list. The `rest` property is itself a function, so to access the
other elements in the list I have to call it and perform the same checks again.

Printing a list, now, is as easy as invoking `each` with the right predicate.

```js
var finite = cons(1, cons(2, cons(3, empty())));
var infinite = repeat(10);

// This prints 1, 2 and 3 on the console
each(finite, console.log);

// This prints an infinite lists of 10's
each(infinite, console.log)
```

The last example in the previous code snippet shows that infinite lists seems
not to be useful by themselves. Who needs an infinite list which always presents
the same element? Even if there are multiple applications for such an infinite
list, it would probably help to define an infinite list which returns a
sequence.

```js
function sequence(start, increment) {
  return function () {
    return {head: start, rest: sequence(start + increment, increment)};
  };
}
```

The `sequence` function returns an infinite list composed of numbers, starting
at `start` and incrementing each number by `increment` at every step. Note how
short the definition of this function is. This is because the definition of the
function is recursive, in the same way that the definition of list is. A
sequence starting from `1` with an increment of `3` can be defined as a list
which has `1` as the head, followed by another sequence starting from `4` with
an increment of `3`.

Even if infinite sequences are beautiful on their own, they are still not
dramatically useful, being my computer a finite piece of hardware. Given a list,
it would be useful to have a function which just returns a part of that list, in
example the first 10 elememts. I can create such a function in a very simple way
and - guess what? - it is another loosely recursive function.

```js
function take(n) {
  return function (list) {
    return function () {
      if (n <= 0) {
        return null;
      }
      
      var current = list();

      if (current) {
        return {head: current.head, rest: take(n - 1)(current.rest)};
      }
      else {
        return null;
      }
    };
  };
}
```

This function is a little bit more complicated than the others, but just because
there are more checks to perform. Moreover, in this case I use two levels of
indirectons. `take` is a function which returns a function which, in turn,
returns another function. This seems a little bit unusual, because the `n` and
`list` parameters can be passed directly to `take`. The reason behind this
indirection will be explained later, when I write more functions like `take`.
For the moment, just take this approach as a given, I'm just too lazy to rewrite
the same function later with a different signature.

The definition of `take` is not really complicated, if you look carefully. If
you pass an empty list to `take`, or if you pass an `n` less than or equal to
zero, you receive an empty list. If the list is not empty, you receive another
list. The head of the list is the head of the list received in input, followed
by a list of `n-1` elements.

This function is really useful if you have an infinite list and you want to
process only the first `6` elements, in example.

```js
var infinite = sequence(1, 3);
var finite = take(6)(infinite);

// This prints the list [1, 4, 7, 10, 13, 20]
each(finite, console.log);
```

I made a lot of progress, but I am still not able to perform very simple
operations on lists, like mapping and filtering. If I can map and filter an
array, I should be able to do it even on lists, which are an extension of the
simple and limited JavaScript array.

```js
function map(mapper) {
  return function (list) {
    return function () {
      var current = list();

      if (current) {
        return {head: mapper(current.head), rest: map(mapper)(current.rest)};
      }
      else {
        return null;
      }
    };
  };
}

function filter(predicate) {
  return function (list) {
    return function () {
      var current = list();

      while (current) {
        if (predicate(current.head)) {
          return {head: current.head, rest: filter(predicate)(current.rest)};
        }

        current = current.rest();
      }

      return current;
    };
  };
}
```

The `map` function is the easiest one. It receives a list in input, and produces
another list whose elements are same as the ones as input, mapped with a
`mapper` function. Even in this case, the function is defined recursively.
Mapping an empty list always returns an empty list. Mapping a non-empty list
returns a new list, whose head is the head of the input list, mapped through
`mapper`, and the rest of the list is the mapped rest of the input list. If this
definition seems difficult, read it again while following the code above.

The `filter` function is a little bit different from every other function I
wrote until now. He is not leaving the input list intact, as `map` is doing.
When you pass a list of `10` elements to `map`, you always receive a list of
`10` elements as output. When you pass a list of `10` elements to `filter`,
instead, you may receive the input list intact, or you may receive a shorter
list because some elements have been filtered out. Heck, you can also receive an
empty list as output! This is because the implementation of `filter` consumes
elements from the input list until an element is found which satisfies the
condition expressed in `predicate`. In this case, the element is returned, and
the rest of the input list is filtered again to remove unwanted elements.

Let's try these new functions on an infinite list. Given the first ten elements
of the sequence `[1, 4, 7, 10, ...]`, I want to know which elements, multiplied
by two, are still less than `30`.

```js
var infinite = sequence(1, 3);
var firstTenElements = take(10)(infinite);
var multipliedByTwo = map(function (x) {return x * 2})(firstTenElements);
var lessThanThirty = filter(function (x) {return x < 30})(multipliedByTwo)

// This prints the list [2, 8, 14, 20, 26]
each(lessThanThirty, console.log);
```

Here I combine some of the functions I wrote to obtain the desired result.
First, I generate an infinite sequence starting from `1` with an increment of
`3`. Then, I select just the first ten elements of the sequence and I save it in
`firstTenElements`. After, I take every element from `firstTenElements`, use
`map` to multiply them by two, and save the result in `multipliedByTwo`.
Finally, I take every element from that list and use `filter` on them to select
only the elements which are less than `30`.

The previous example shows the main advantages of lazy computation. No list
processing happens if I remove the final call to `each`. If there is no one
consuming the list of numbers, then there is no point in performing mapping and
filtering. On the other hand, because of lazyness, I can remove the call to
`take` and perform mapping directly on the infinite sequence. In this case,
everything would work as expected: even if the list is infinite, I don't have to
save the intermediate results anywhere. Because no intermediate results are ever
saved, but instead computed on the fly, I can process infinite lists in a finite
amount of memory.

The final touch to the previous code would be some helper function which allows
me to concatenate an arbitrary amount of calls to `take`, `map` and `filter` to
generate reusable manipulation functions for lists. For this purpose, I will
create a `pipe` function which is able to connect multiple operations on lists
and invoke them easily.

```js
function pipe() {
  var fs = arguments;

  return function (list) {
    var i, result = list;

    for (i = fs.length - 1; i >= 0; i--) {
      result = fs[i](result);
    }

    return result;
  }
}
```

The `pipe` function takes an unspecified amount functions which, given a list,
return a new list as output, and returns a new function which is the combination
of the original ones. For this reason, I can rewite the last example by
factoring out some functions and rewrite everything using a call to `pipe`.

```js
var input = sequence(1, 3);

var takeTen = take(10);
var multiplyByTwo = map(function (x) {return x * 2});
var filterLessThanThirty = filter(function (x) {return x < 30});

var process = pipe(filterLessThanThirty, multiplyByTwo, takeTen);

// This prints the list [2, 8, 14, 20, 26]
each(process(input), console.log);
```

This makes everything more readable. It's easy to understand that a call to
`pipe(C, B, A)` means that creates a new function which invokes `C(B(A))`. In
other words, `pipe` is creating a function receiving the same input as `A` and
returning the same output as `C`. Every time a new element is available from the
input list, this element is first process by `A`, the result of `A` is processed
by `B` and, finally, the result of `B` is process by `C`, which will compute the
final output. If you are wondering, yes, this is exactly how function
composition works.

I could factor out the list processing operations in `takeTen`, `multiplyByTwo`
and `filterLessThanThirty` because I wrote `take`, `map` and `filter` in a
special way. Every list processing function, in fact, doesn't process the list
directly, but instead returns a new function which always has the same
signature. If you write each list processing step as a function which receives a
list and returns a new list, then it is easier to concatenate them in a new,
bigger step using `pipe`.
