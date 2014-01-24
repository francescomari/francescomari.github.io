---
layout: post
title:  "Using the Sling dynamic classloader"
date:   2013-09-12
---

When you write a bundle for Sling, or for another OSGi based application, you are faced with a dynamic environment. Bundles and classes can come and go, and you should keep track of events emitted by the framework to carefully update your point of view on the state of the system. Moreover, a bundle can’t usually get references to classes from unknown, unimported packages.

## What about dynamic imports?

The `DynamicImport-Package` manifest header allows a bundle to import an unspecified number of packages. You can use wildcards to import a well defined set of sub-packages, like `com.foo.*`, or you can use a simple wildcard to allow imports from each and every package in the system.

This header looks very handy, but its usage is not reccommended. Why? In the first place, it bypasses the normal requirement resolution of the OSGi framework, which will not be able to know in advance if every dependency of the bundle can be satisfied before it is activated. In the second place, the behaviour of `DynamicImport-Package` is not so dynamic. When your bundle asks for a class, the framework looks for it in each deployed bundle; if the class is not found your code must be prepared to handle a `ClassNotFoundException`. If the class is found, then it is returned to your bundle and, at the same time, the framework builds a wire between your bundle and the exporting bundle. When this wire is established, subsequent requests for the same class will be satisfied by the same exporting bundle.

In conclusion, the `DynamicImport-Package` header is dynamic in the beginning, but it turns into a normal import as soon as the framework is able to resolve the requested class. You should avoid this header unless you don’t have another way to solve your problem.

## Introducing Sling Dynamic Classloader

Sling has one bundle, `org.apache.sling.commons.classloader`, which is a general implementation of a really dynamic classloader. It’s general because it isn’t tied to any Sling API, and it can be reused in other OSGi applications; it’s dynamic because it listens to events emitted by the framework to dynamically import classes from the existing set of deployed bundles.

To use the Dynamic Classloader you have to get an instance of the `DynamicClassLoaderManager` service. This service acts like a factory for dynamic classloaders. Its only method, `getDynamicClassLoader()`, returns a `ClassLoader` object which you can use to dynamically load classes from deployed bundles.

The returned `ClassLoader` object will work as a normal class loader, but with one difference: it can become invalid if certain conditions are met.

The first condition is if the bundle calling `getDynamicClassLoader()` is deactivated. This capability is useful if you plan to create a class loader in your bundle and pass it around the system: if your bundle is updated or unregistered, the class loader you have passed around will become invalid.

The second condition is if a bundle is started. If a dynamic class loader exists which failed to load a class from that bundle, every dynamic class loader in the system will become obsolete and will refuse to load other classes. The reason for that is that, since the exporting bundle is started again, there’s a chance that the missing class is now available.

The third condition is if a bundle disappears from the system because it enters the unresolved state. If one dynamic classloader exists which used that bundle to load a class in the past, every dynamic classloader in the system must now become obsolete, because that class doesn’t exist in the framework anymore.

When a class loader becomes invalid, class loading will stop working in a graceful way. Everytime an invalid classloader is used, an error message will be logged and methods like `getResource()`, `loadClass()`, etc. will simply return `null`.

## Inside the Sling Dynamic Classloader

The Sling Dynamic Classloader is very simple to understand. This is a good OSGi excercise, so pay attention to the following steps.

1. The `org.apache.sling.commons.classloader` bundle has an activator class. The activator performs some interesting tasks, such as getting a reference of the Package Admin service, listening for changes in the set of bundles deployed in the framework, and registering a service factory called `DynamicClassLoaderManagerFactory`.
1. The service factory is in charge of creating new instances of the `DynamicClassLoaderManager` service for each bundle in the system. Other than that, the service factory also maintain two important pieces of information: the set of bundles where classes are loaded from and the set of packages which failed to load. As you will see, this data is shared by every dynamic class loader.
1. The implementation of the `DynamicClassLoaderManager` is not that interesting. Really. It’s just a wrapper around the `PackageAdminClassLoader` class, which performs the bulk of the work. For each instance of `DynamicClassLoaderManager` a new instance of `PackageAdminClassLoader` is created.
1. The `PackageAdminClassLoader` is the real core of the dynamic class loading. It is a subclass of `ClassLoader` which searches for classes and resource in the current bundle first, and in every deployed bundle as a second choice. The `PackageAdminClassLoader` will also take care of updating the global information stored in `DynamicClassLoaderManagerFactory`. Every time a class is successfully loaded, the bundle where the class is loaded from is stored in the factory; every time a class is not found, the package of the requested class is stored in the factory too.

The interaction between `DynamicClassLoaderManagerFactory` and `PackageAdminClassLoader` is very important. Thanks to the information provided by each instance of `PackageAdminClassLoader`, the activator of the `org.apache.sling.commons.classloader` bundle is able to decide if and when to invalidate every dynamic class loader in the system.

Pay attention to one final aspect. If a bundle changes in the system, and even one dynamic classloader exists which loaded class from it, then every dynamic classloader will be invalidated. This is very important, because it represents the dynamic aspect of this classloading strategy.
