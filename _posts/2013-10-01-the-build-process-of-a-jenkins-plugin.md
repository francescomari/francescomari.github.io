---
layout: post
title:  "The build process of a Jenkins plugin"
date:   2013-10-01
---

Jenkins allows you to extend the system by writing new plugins, which can be installed into the CI server manually or automatically. Tooling provided with Jenkins helps in writing, validating and testing new plugins but the build process of a Jenkins plugin is never described in detail.

## The easy way

The hard and fast way to bootstrap your project is to use the latest Maven archetype for a Jenkins plugin. This will automatically create the project folder structure and the POM file which will take care of building your plugin. You don’t need to investigate more, because the basics of building and testing your plugin is described in the [Plugin Tutorial](https://wiki.jenkins-ci.org/display/JENKINS/Plugin+tutorial), which is part of the official documentation.

## Understand the build process

The feature that I like (and dislike) more about Maven is the possibility of adding new packaging types. Each packaging type has the possibility to add one or more Maven plugin executions to the build process. So, every time you use a packaging type in your POM, you get a basic configuration for free. At the same time, though, you lose a little bit of control on your build process, because to actually know what’s going on you have to dig into the build extensions declared in the POM and read the descriptor file which declare the packaging type you are using.

Jenkins relies on a custom packaging type. If you are writing a new plugin, your POM file should have a packaging type of hpi. The plugin which declares the packaging type is the [Maven HPI Plugin](https://github.com/jenkinsci/maven-hpi-plugin), and it is documented [here](http://jenkins-ci.org/maven-hpi-plugin/). Unluckily, the official documentation doesn’t mention what happens when the custom packaging type is used.

Let’s try to dissect the build process of a Jenkins plugin.

1. In the first place, some preconditions are checked. To correctly build a plugin you need Java 6 or later, and your plugin must support Jenkins 1.420 or later. To detect the version of Jenkins your plugin supports, your dependencies are scanned. You must declare a dependency on `org.jenkins-ci.main:jenkins-core` or on `org.jvnet.hudson.main:hudson-core`.
1. Resources are processed and classes are compiled by Maven as usual, by using the Maven Resources Plugin and the Maven Compiler Plugin.
1. The compiled classes are scanned for restriction violations. To achieve this goal, another Maven plugin is used, the [Access Modifier Checker Plugin](https://github.com/kohsuke/access-modifier). The general idea is that your code can declare some restrictions, such as to never use a class as a base class, or to never instantiate a class of a given type, and the Access Modifier Checker Plugin will check that your bytecode fulfills the requirements.
1. Some test classes are generated to perform static syntactic checks on the Jelly files included in the Jenkins plugin.
1. An HPI file is created which will be used during the testing process.
1. If your plugin depends on other plugins, those dependencies are copied in the test dependency folder.
1. Test are executed by the Maven Surefire Plugin.
1. The plugin is packaged in two versions. The first version is a JAR file, which is used as a dependency when another plugin requires the current plugin. The second version is an HPI file, which is an exploded web application suitable to be installed into a Jenkins server.
1. The installation and deploy phases, if invoked, are managed by Maven in the usual way.

Knowing what happens in your build process is always useful. Please remember, when you face a new project read (and understand) the POM file first. You have been warned.

