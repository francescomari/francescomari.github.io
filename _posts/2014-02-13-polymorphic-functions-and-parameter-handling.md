---
layout: post
title:  "Polymorphic functions and parameter handling"
date:   2014-02-13
---

JavaScript functions are very flexible when it comes to receiving parameters.
Actually, functions can take any number of parameters via the `arguments`
special variable. It is up to the developer to support different combinations of
parameters by checking their type and handling them correctly. Good news is that
this logic can be encapsulated and reused to easily build polymorphic functions.

Let's start with an example of a poorly written function, which manually checks
for the types of the input parameters. The function identifies optional, non
provided parameters and gives them default values if possible.

```js
function provideSum(a, b, f) {
  if (f === undefined) {
    f = b;
    b = 0;
  }

  if (f === undefined) {
    f = a;
    a = 0;
  }

  if (!f) {
    throw new Error('no function provided!');
  }

  f(a + b);
}
```

What the heck is going on here? It is not immediate to see which parameters the
function requires, which ones can be omitted and which default values are
automatically provided by the implementation. Most of the code is just about
shuffling parameters around, distracting the developer from the real purpose of
the function.

What I want to achieve is a simple way to define how the behaviour of the
function changes when different parameters are passed. Something like the
following:

```js
var provideSum = when(
  matches('function', function (f) {
    f(0)
  }),

  matches('number', 'function', function (n, f) {
    f(n)
  }),

  matches('number', 'number', 'function', function (a, b, f) {
    f(a + b)
  })
);
```

To make this solution work I have to implement a couple of functions, `matches`
and `when`. The first function I'm going to implement is `matches`, which checks
if an array of arguments fulfills the requirements expressed by the user. If the
parameter values match the specification, a function (which is the last
parameter of the call to `matches`) is returned. Otherwise, `matches` simply
returns `null`.

```js
function array(args) {
  return Array.prototype.slice.call(args);
}

function matches() {
  var types = array(arguments).slice(0, -1);
  var fn = array(arguments).slice(-1).pop();

  function matcher(args) {
    if (args.length !== types.length) {
      return null;
    }

    var match = args.every(isCorrectInstance);

    if (match) {
      return fn;
    }

    return null;
  }

  function isCorrectInstance(arg, i) {
    var type = types[i];

    if (typeof type === 'string') {
      return typeof arg === type;
    }

    if (typeof type === 'function') {
      return arg instanceof type;
    }

    return false;
  }

  return matcher;
}
```

The first check the function executes is if the array of values `args` contains
the same number of elements of the array of types. If this is not the case, it
can immediately return `null`, since we know that the user provided a wrong
number of parameters. The second check is on the type of the arguments. The
`isCorrectInstance` function , which implements the type matching logic, accept
two types of parameter specifications:

- a string: in this case, the `typeof` operator is used on the argument, and the
  result is then compared with the value provided to `matches`.

- a function: this allows to define parameter types in term of constructor
  functions. The match will be satisfied only if an object built by the given
  constructor is provided. The `instanceof` operator is used to perform this
  kind of check.

The last and missing piece is `when`, which generates the final function. The
logic of `when` is quite simple, because it only has to go through every
matcher, invoke them, and call the first non `null` function.

```js
function when() {
  var matchers = array(arguments);

  function filtered() {
    var args = array(arguments);

    var fn = matchers.reduce(matches, null);

    function matches(result, matcher) {
      return result ? result : matcher(args);
    }

    if (fn) {
      return fn.apply(null, args);
    }

    throw new Error('invalid arguments');
  }

  return filtered;
}
```

This approach has different benefits compared to the original implementation. In
the first place, the  code is highly reusable. The logic about parameter
checking is encapsulated in a small, simple and reusable piece of code.
Moreover, the implementation of `provideSum` is cleaner because each special
case is handled in different functions. Each function can now focus only on the
logic because it can assume that parameters will always be there and will always
be of the correct type. This helps to improve the overall quality of the code,
making your implementation more stable and readable.
