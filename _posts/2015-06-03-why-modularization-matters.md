---
layout: post
title:  "Why modularization matters"
date:   2015-06-03
---

I recently started working on a big, monolithic project. In my opinion, this project is a good candidate for an aggressive modularization effort. Let's imagine that someone is arguing against this effort, and let's analyze his statements. Are all of these statements true? Can modularization really provide some benefits to users and developers?

Before I start, I have to give some context. The project I'm talking about is about 150K lines of code, with an additional 75K lines of tests. The project is split in some well-defined layers. Some layers expose a plugin system that allows new functionalities to be easily implemented and integrated. The project also contains many implementations of layers and plugins. At run-time, it is necessary to choose a combination of layer and plugin implementations for this project to work. The project is written in Java and it's deployed as a monolithic OSGi bundle composed of many, many packages. A big part of these packages are part of the public API of the bundle.

Every section below, except the conclusions, starts with an argument against modularization. I often use the project I described as a reference for my counterarguments, but the prose is general enough to be used in other contexts. Let's get started!

## The project is already modularized

*The project is obviously already modularized. The layers form a first set of boundaries, and plugins make everything clearer. At run-time, just instantiate the concrete implementations of plugins and layers, wire them together and you have a running system in no time!*

This statement is misleading. In theory, interfaces and implementations for layers and plugins are contained in different packages. In practice, the situation is not so easy. With time, deadlines and pressure, some abstraction started leaking. Some implementation detail was shared between packages that shouldn't be related with each other. Many TODOs were added to the code to remind the future maintainers that the workarounds should be removed in the future.

Developers are humans, management is cruel and deadlines are just another pain in the butt. Hammering workarounds in the projects is a natural reaction of stressed developers, but it could be prevented if every logic layer was contained in a different project. If a class is not visible from one bundle, it's because that class is an implementation detail. If a developer wants to expose that class, it has to change the exposed API of the bundle. If proper semantic versioning and tooling support is in place, doing so would fail the build, and make aware the stressed developer - and the rest of the team - that something is going wrong.

Having many independent bundles also makes everything easier to understand. Instead of wrapping his head around a huge code base, a new contributor can analyze a bunch of class at a time, where each group of class belongs to the same bundle. The human mind is limited, and it is always better not to think about too many concept at once, but split the reasoning in more manageable pieces.

## Use rules to check inter-package dependencies

*If the problem is leaking abstractions from one package to the other, an additional build step can be added to check for violations. A set of rules checking inter-package dependencies is way easier to maintain than many independent bundles!*

Failing the build, and failing it fast, is always a good thing. Trading proper modularization for a set of rules to check for inter-package dependencies is not. Let's assume that the project is not modularized and we have this new check at build time. There are some possible rules can be implemented by such a check:

- allow export rules: a class C is allowed to be exported. This means that every developer that wants to export a class from a package has to make the class public **and** modify the set of rules to make clear that this is allowed. This is like stating the same thing twice, and in the long run it may be very annoying.

- deny export rules: a class C is not allowed to be exported. I assume that this is a sensitive default. By default, no class should be exported. This, of course, forces the developers to explicitly state which classes should be exported instead. See the previous point.

- allow import rules: a package P is allowed to be imported from another package Q. Since every package belongs to the same bundle, it's important to check that a package P that is an implementation detail of package Q is imported only by package Q. If this case is not checked, it may be that some implementation details of package Q are leaked. This kind of rule becomes really annoying to maintain when you have a utility package that is imported many times. You could use wild-cards for that, but everybody knows that using wild-cards and white-list-style rules is asking for trouble.

- deny import rules: a package P is not allowed to be imported from another package Q. Once again, this should be a sensitive default. This means that using a rule like the one described in the previous step is mandatory for every inter-package import.

In the project I described above there are about 420 occurrences of "public class" or "public static class", and about 4000 occurrences of imports of classes from the same project. I don't know how many of them should be there, but let's be very optimistic and say that only half of them should be mapped by rules. Would you really feel happy to maintain a set of rules this big? It may just be my personal taste, but I would prefer to maintain a dozen of independent bundles than such a huge list of rules.

## The build is not reproducible

*With a big number of project it is really difficult to reproduce a particular build. There are too many possible combinations of bundles to be sure that everything is working as expected. We loose support from the compiler.*

Being the project a single, monolithic bundle, developers are used to make some small changes, recompile everything and hope that the compiler doesn't shout at them. This approach is not so bad because the compiler is smart and should be trusted. Modularization just brings this concept to a new level of reliability.

Modularization usually goes hand-in-hand with semantic versioning. If you are not familiar with it, semantic versioning gives a very precise meaning to a version number. By looking at two version numbers, you can immediately tell if the new version is just fixing some bugs, adding new features or severely breaking backwards compatibility. If you split your project in different bundles, you don't add snapshot dependencies between them. Instead you define dependencies on version ranges, where a bundle declares that it can work with many versions of the same dependency as long as these versions don't break the expected level of compatibility.

When you are working in a specific bundle, you can still trust the compiler as usual. If you break something, the compiler still shouts at you. The new level of reliability I mentioned above is that now you can check if you are breaking the users of your bundle, too. If your bundle adopts semantic versioning, it is possible to compare at build time the exported API of the new version of your bundle with the old one. If you are breaking the contract defined by semantic versioning, the build with fail.

You can't have these checks if everything is in the same monolithic bundle. Sure, if you add a method to an interface, something will break the build. But there is more to your project than the code you are writing, and you also have to think about the users of your code. You can't be a selfish developer and just think about how simple your workflow should be.

## We need flexible deployments

*If we have everything in one bundle, we can deploy all the fixes at the same time. Having many bundles just makes everything more complicated, because you have too many deployment units.*

Here the developer is making the assumption that every user of the project wants all the fixes included in the new version. Worse than that, the developer is assuming that everyone is using the project in its entirety. In example, the project described above is exposing many implementations of layers, but it is very common that a typical deployment of this project just uses a very small subset of them.

If you develop every layer as an independent bundle, you can release it independently from the rest of the project. If you have to give a very urgent fix to a customer, you don't have to release the whole project. You can focus on a limited amount of bundles, cutting a new release of only the impacted ones. If you are also using semantic versioning, you can make clear that the new version is a bugfix by using an appropriate version number. Users of your bundle will be more confident of deploying it if they now that it is not breaking backwards compatibility.

When it comes to deployments, handling many small components is easier than managing a monolith. It is impossible for the deployer to select just a subset of the layers exposed by your project, if they are all bundled in the same deployment unit. By deploying your project as a monolithic bundle, you are forcing an all-or-nothing deployment model on your users.

Moreover, it is easy to aggregate many small deployment units into a bigger one than to split a monolithic deployment unit into smaller ones. If you have many small bundles, you can embed all of them into an OSGi subsystem - or any other equivalent mechanism. If you have a monolithic bundle, you have to refactor it first.

## Conclusions

I couldn't possibly enumerate all the nonsensical arguments against modularization said by people all over the world. If you think that I left an important argument out, let me know. 

I also recognize that modularization, as every engineering technique, has benefits and drawbacks. I don't think that modularization is a cure-all, but I strongly believe that is a handy tool to manage complexity in oversized projects.

Sometimes developers argue against modularization because they are afraid to change. To these people I say that making users and contributors happy by providing a clean and modularized code base is way more important than holding to a comfortable, sub-optimal development workflow.

