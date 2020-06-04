---
layout: post
title:  "Option builders in JavaScript"
date:   2014-01-13
---

Using option objects in JavaScript is a common pattern for methods requiring a
lot of different parameters. This technique allows a method to stay flexible and
evolve gracefully when new parameters are added or an old parameter changes name
or meaning.

Creating option objects for these methods, though, is a different story. The
task is often simple and repetitive and it is often overlooked during the day-
to-day development. It is common to see code like that spread over your program:

```
function searchUsers(user, password, name, age, done) {
  var options = {
    auth: {
      user: user,
      password: password
    },
    url: 'http://api.acme.com/users/search',
    method: 'POST'
  };  

  options.form = {};

  if (name !== null) {
    options.form.name = name;
  }

  if (age !== null) {
    options.form.age = age;
  }

  request(options, done);
}
```

It is pretty clear that this method is performing a POST request to a well known
URL to search for users, optionally providing a name and an age to narrow the
search results. Authentication information is also provided in the `option`
object to authenticate to the remote service.

There are different problems with the previous example:

- The `option` object is created and modified manually. This approach is
  sensitive to different kinds of error, like misspelling an option name.
- Cross-cutting concerns are repeated without any generalization. Setting the
  authentication data should be encapsulated in its own method. You don't want
  to change the whole application if you later decide to change your
  authentication strategy.
- Conditionally adding parameters to the option object is tedious and fills the
  method with a lot of `if` statements.
- The workflow to build an option object is not standardized, but shows small
  and subtle differences every time it is executed. This is what happens if you
  manually build an option object.

## A first generalization

Tho solve the first problem you could encapsulate option assignment in different
functions. Every function receives an option object and modifies it, as shown in
the following code snippet.

```
function setAuth(options, user, password) {
  options.auth = { user: user, password: password };
}

function setUrl(options, url) {
  options.url = url;
}

function setMethod(options, method) {
  options.method = method;
}

function addParameter(options, name, value) {
  var form = options.form || {};
  form[name] = value;
  options.form = form;
}
```

This brings our original method to the next level.

```
function searchUsers(user, password, name, age, done) {
  var options = {};

  setAuth(options, user, password);
  setUrl(options, 'http://api.acme.com/users/search');
  setMethod(options, 'POST');
  
  if (name !== null) {
    addParameter(options, 'name', name);
  }
  
  if (age !== null) {
    addParameter(options, 'age', age);
  }
  
  request(options, done);
}
```

This solves the very first problem in our bullet list. You avoided every
possible spelling mistake during the construction of the `option` object.
Moreover, if an option parameter will change its name, you have to modify your
code in only one place.

You can apply an additional improvement to the previous code by eliminating the
two `if` statements. It is as easy as writing a more specialized function to
change the option object.

```
function addParameterIf(options, condition, name, value) {
  if (condition) {
    var form = options.form || {};
    form[name] = value;
    options.form = form;
  }
}
```

This makes the function code shorter and clearer.

```
function searchUsers(user, password, name, age, done) {
  var options = {};

  setAuth(options, user, password);
  setUrl(options, 'http://api.acme.com/users/search');
  setMethod(options, 'POST');
  addParameterIf(options, name !== null, 'name', name);
  addParameterIf(options, age !== null, 'age', age);

  request(options, done);
}
```

## Stretch it a little bit more

Our function is in a better shape than before, but it still feels boring to
write. Well, nobody said that your job should be funny, but it should at least
by as easy as possible.

I assume that you have a lot of functions in your codebase very similar to the
previous one. If they are so similar, there should be a way to generate them.
The first step in this direction is to generalize the creation of an option
object. At the moment this is not possible because each helper function you
created has a different signature. While this can be good for you, it is surely
a bad thing for your computer.

What if each helper function had the same semantic?

```
function setSomething(options) {
  // Do something
  return options;
}
```

If every helper function was written like this, you could execute all of them in
order and pass the `options` parameter from one function to the other, until all
of them are executed.

You can change your previous helper function to generate functions of this type,
instead of directly accessing the options object. They will just create a
closure for other functions which perform the actual work. Let's give it a shot.

```
function setAuth() {
  return function (options) {
    options.auth = getAuth();
    return options;
  };
}

function setUrl(url) {
  return function (options) {
    options.url = url;
    return options;
  };
}

function setMethod(method) {
  return function (options) {
    options.method = method;
    return options;
  }
}

function addParameterIf(condition, name, value) {
  return function (options) {
    if (condition) {
      var form = options.form || {};
      form[name] = value;
      options.form = form;
    }
    return options;
  };
}
```

Note that I changed the logic of the `setAuth()` function. This implementation
require an external method to provide the `auth` parameter to be set in the
`option` object. In this way, you can throw out the user name and password from
the parameter list of your functions. By applying the previous modification, you
will end up with the code shown below.

```
function searchUsers(name, age, done) {
  var operations = [
    setAuth(),
    setUrl('http://api.acme.com/users/search'),
    setMethod('POST'),
    addParameterIf(name !== null, 'name', name),
    addParameterIf(age !== null, 'age', age)
  ];

  var options = operations.reduce(toOptions, {});
  
  function toOptions(options, operation) {
    return operation(options);
  }
  
  request(options, done);
}
```

Please note that the operations array is a sequence of functions, and each of
them has the same signature: they all receive an option object in input and
return an option object as output.

You use the `Array.reduce()` method to invoke every function in order, and
incrementally construct the final `option` object. As shown from the invocation
of `reduce()`, the `option object is initially an empty object.

## Less work for everyone

Given the improvements in the previous section, can you really generalize our
function to generate it at runtime? In the end, a function like that is only a
sequence of steps which must be performed to create an option object, and it
always performs a call to the request() method with the the option object and a
callback. Moreover, the callback is always the last parameter of the function.

The only information left in the original function, at the end of the
transformation, will be this:

```
function searchUsers(name, age) {
  return [
    setAuth(),
    setUrl('http://api.acme.com/users/search'),
    setMethod('POST'),
    addParameterIf(name !== null, 'name', name),
    addParameterIf(age !== null, 'age', age)
  ];
}
```

What you have to write is a generator which takes something like that in input
and generates a working function as output. Writing such a generator, as shown
below, is a very simple task which requires only a little bit of juggling with
the `arguments` special variable.

```
function createFullMethod(partialMethod) {
  return function () {
    var args = Array.slice.call(arguments);
    var done = args.pop();
    var operations = partialMethod.apply(null, args);
    var options = operations.reduce(toOptions, {});

    function toOptions(options, operation) {
      return operation(options);
    }

    request(options, done);
  };
}
```

The `createFullMethod()` function creates a new function. The arguments of this
function will be evaluated dinamically, and the number of arguments can be
variable. Let's go through the steps performed by this function:

1. You convert the `arguments` special variable to a full array by using the
   `Array.slice()` method.
1. You extract the `done` callback from the end of the arguments. This is a
   convention which enforces every function to have a callback as last
   parameter. The `args` variable now contains the remaining parameters passed
   to the function.
1. You call `partialMethod` with the rest of the arguments to obtain a list of
   operations.
1. You `reduce()` the operations to have a full option object.
1. You perform the request and pass the `done callback to be called when a
   response is received.

This final improvement means that you can generate other functions very similar
to your original one with a minimum effort. Let's see how you would now rewrite
the original function and how we would use the same API to also create a
different remote function.

```
var searchUsers = createFullMethod(function (name, age) {
  return [
    setAuth(),
    setUrl('http://api.acme.com/users/search'),
    setMethod('POST'),
    addParameterIf(name !== null, 'name', name),
    addParameterIf(age !== null, 'age', age)
  ];
});

var readLatestStats = createFullMethod(function () {
  return [
    setUrl('http://api.acme.com/stats/latest'),
    setMethod('GET')
  ];
});
```

## Conclusions

If, while reading this article, you thought that this approach is overkill, your
view was probably limited to the only function you refactored. The described
approach is useful if you have plenty of functions doing almost the same thing
with some small variations, because you have a common API to describe the parts
these functions are composed from.

In particular, you should focus on how we represented a function as a list of
operations. The moment you treat functions as first class objects which you can
store, manipulate, and combine, your code will be automatically become simpler,
your abstractions stronger and your flexibilty endless.
