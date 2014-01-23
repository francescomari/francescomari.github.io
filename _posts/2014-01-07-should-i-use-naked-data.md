---
layout: post
title:  "Should I use naked data?"
date:   2014-01-07
---

I’m not the first one to write about this topic, and someone before me already wrote books about it. The reason behind this article is that I’m facing some design issues in a piece of code I’m writing, and probably explaining my ideas will help myself more than anyone else.

## The OOP approach

I come from a Java background, and when I write some JavaScript code it feels natural to model my objects using an OOP approach. In the first draft of my code I usually end up with something like that:

{% highlight js %}
function Person(name) {
  this.name = name;
  this.age = null;
}

Person.prototype.getName = function () {
  return this.name;
};

Person.prototype.getAge = function () {
  return this.age;
};

Person.prototype.setAge = function (age) {
  this.age = age;
};
{% endhighlight %}

There is nothing wrong about this approach, especially if you plan to attach some real behaviour to the prototype later. At the moment, though, I see two main issues with this solution.

First, creating a prototype to act as a container object for your data seems a little bit overkill. This forces you to instantiate your object using the `new` keyword and to access the underlying data with ugly getters and setters. This makes your code more verbose both in the definition of the prototype and in the code using the prototype itself.

Second, defining getters and setters doesn’t prevent anyone from accessing the state of the object directly. There is no enforcement other than pure convention. Warnings in documentation (if you write any) are not enough to force the users of your code to behave properly.

## Exploiting closures

Luckily, JavaScript offers a feature called closures. This allows you to create a real private space to save the internal state of the objects. Instead of using classes and constructors, you can implement the creation logic of your object as a plain function:

{% highlight js %}
function createPerson(name) {
  var age = null;

  return {
    getName: function () {
      return name;
    },
    getAge: function () {
      return age;
    },
    setAge: function (value) {
      age = value;
    }
  };
}
{% endhighlight %}

There is no way to access the real values of the name and the age of the newly created person, and access restrictions and validation policies can be implemented in a more consistent way.

## The prototypal way

The previous approach still leaves you to with an ugly API based on getters and setters. It would be really cool to access the data directly, like they were simple properties of an object! This is possible, according to the ES5 specification, using the `Object.create()` method. The previous example, when converted to the prototypal approach, looks like this:

{% highlight js %}
function createPerson(name) {
  var age = null;

  return Object.create(null, {
    name: {
      enumerable: true,
      value: name,
      writable: false
    },
    age: {
      enumerable: true,
      get: function () {
        return age;
      },
      set: function (value) {
        age = value;
      }
    }
  });
}
{% endhighlight %}

Now you fulfilled every requirement: your object has a real private state thanks to a closure, and accessing your data is as simple as reading or writing a property on an object. Your validation code, if any, can run inside a setter function which will be called every time a value is assigned to the corresponding property. But even if this implementation is complete according to your original requirements, it still has some disadvantages.

First, the code has become less readable. It’s not immediately clear that you are defining properties of an object, and the property definitions use different kind of property descriptors, making the code less intuitive. An inexperienced JavaScript developer would probably find this code difficult to understand at first. The API you are using, even if it is defined by the EcmaScript standard, is not something you see every day.

Second, this implementation could have a big performance impact on your code. At the moment of this writing, creating an object via Object.create is one of the slowest options you have.

## Undress your data

I have to admit that I’m not a big fan of the approach described in the previous section, unless it’s not strictly required. I would better prefer to use naked data:

{% highlight js %}
var person = {
  name: 'Benny',
  age: 65
};
{% endhighlight %}

Your data is there and it’s readable. But what about the access restrictions? If a piece of code should not read your data, you can always create a restricted representation of it, in the same way a database view restricts the access to the columns in a table.

{% highlight js %}
var restricted = _.pick(person, 'name');
{% endhighlight %}

In the previous example I used a method defined in the Lo-Dash library for clarity. The `pick()` method creates a shallow clone of the `person` object, filtering just the `name` property. This produces a new object which is a restricted version of the original one, without the sensitive properties which you don’t want to pass around.

Validation is a problem which can be addressed in a different way. Instead of validating the correctness of a value each and every time you assign it to a member variable, you should actually restrict where validation takes place. If you are writing a Node module, in example, you have a well defined set of access point to your module functionalities. It is sufficient to validate your input data once, and the internals of your module can always treat the data assuming that is valid. If you are not sure that the internal implementation of your module treats the data correctly, maybe you should consider unit-testing your code more thoroughly.

## A step further
Having plain data objects enables a different way of thinking about your software. Instead of focusing about objects, classes and behaviour, you can instead think about how the data can be transformed to achieve your goals.

I don’t want to bother you with functional programming concepts, because I prefer to show you how easy can be to manipulate plain data using a little bit of functional magic.

{% highlight js %}
function getName(person) {
  return person.name;
}

function names(people) {
  return people.map(getName);
}

var people = [
  {name: 'John', age: 39},
  {name: 'Paul', age: 23}
];

names(people); // -> ['John', 'Paul'] 
{% endhighlight %}

This approach, of course, works even if you don’t adhere to the naked data philosophy. You have to admit, though, that it looks very natural to write this kind of code if you only have to bother about plain data. The previous approach can also be generalised by providing more flexible functions. What happens if you encapsulate the attribute access code in its own function?

{% highlight js %}
function get(property, object) {
  return object[property];
}

function names(people) {
  return people.map(get.bind(‘name’));
}
{% endhighlight %}

In the previous example you have a very general function, get, which can be reused in a very broad set of different contexts. As I told you before, this is not a direct consequence of naked data, but it feels really natural to think about data transformations when you don’t bother about classes and private state.

## Is it really useful?

The short answer is: it depends. If you are writing a simulation system, it would make sense to organise your code in objects interacting with each other, and OOP feels like the natural choice. In my experience, though, I find myself writing applications which usually read data, apply some transformations to it, and write it somewhere else. In this case, it doesn’t make sense to have classes modelling your data, because you want to put most of your effort in the code which transforms it.

In conclusion, this article just describes a technique I find useful in some contexts, and it’s not meant to be the solution to every problem of each JavaScript developer. The first thing to do before choosing any programming technique is to reason about your problem, and only after you can decide the right approach to use.
