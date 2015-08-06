---
layout: post
title:  "A plugin system in Node"
date:   2013-05-16
---

When developing Node applications is often needed to implement a plugin system.
You want to do this to enable other developers to provide features to your
software without cluttering your code base, and to define a clear separation
between core and externally-provided features.

In this post I will analyze the solution implemented by Grunt, and I will detail
how to leverage the structure of Node modules and NPM packages to implement your
own plugin system.

## What is a Node module

The simplest version of a Node module is .js file, but Node uses a comprehensive
[resolution algorithm](http://nodejs.org/api/modules.html#modules_all_together)
to load code stored in different layouts. In a nutshell, if a module at path
`/home/user/project/client.js` invokes a `require("dep")`, the following
directories are searched for the module, in the provided order:

```
/home/user/project/node_modules
/home/user/node_modules
/home/node_modules
/node_modules
```

For each directory `<dir>` in the previous list, the following files are
checked, in the provided order:

```
  <dir>/dep
  <dir>/dep.js
  <dir>/dep.node
  <dir>/dep/package.json
? <dir>/dep/<main>
? <dir>/dep/<main>.js
? <dir>/dep/<main>.node
  <dir>/dep/index.js
  <dir>/dep/index.node
```

The only special case is represented by the `package.json` file. If this file
has a `"main"` field, the content of this field is treated as a file location,
relative to the directory containing `package.json`. This explains the entries
with a `?` in the previous list. Those locations are only checked if
`package.json` has a valid `"main"` field. The content of this field will
substitute the `<main>` placeholder in the previous paths.

## What is an NPM package

An NPM package is a folder containing resources and described by a
`package.json` file. There is a well made [interactive
guide](http://package.json.nodejitsu.com/) for `package.json` files out there,
please check it out to have a quick grasp of what kind of information can be
attached to an NPM package.

Node and NPM are designed to work side by side. If you install an NPM package,
the npm tool will place it inside a `node_modules` folder; if the root of the
package contains an `index.js` file or if its `package.json` file has a valid
`"main"` attribute, the package will be eligible for resolution by Node.

Even if, in my humble opinion, a Node module and an NPM package are not strictly
the same entity, an NPM package is a specialization of a Node module and
therefore can be used interchangeably as a Node module.

## How Grunt loads plugins

Grunt has a full featured plugin system to load new tasks. Tasks can be loaded
from a directory or from locally installed NPM packages. To load new tasks
inside your `Gruntfile.js` you execute code similar to the following:

{% highlight js %}
grunt.loadTasks(folder);
grunt.loadNpmTasks(packageName);
{% endhighlight %}

The code which loads new tasks can be found in `lib/grunt/task.js`. This module
contains the implementation of the methods `loadTasks()` and `loadNpmTasks()`
and relies upon basic task features provided by the module `lib/util/task.js`.
Let’s analyze first how Grunt loads new tasks from files. The bulk of the logic
is contained in the `loadTask()` method and, in particular, in the following
section of code:

{% highlight js %}
fn = require(path.resolve(filepath));
if (typeof fn === 'function') {
  fn.call(grunt, grunt);
}
{% endhighlight %}

In this snippet of code, `filepath` is an absolute path to a file. Grunt uses
the standard `resolve()` function to load and evaluate the content of the file,
and to return whatever the module exports. In this case, Grunt expects a module
to export a single function, as you can see from the check performed by the if
statement. The exported function `fn` is subsequently called with grunt as both
`this` object and first argument.

The previous case was the simplest one. A little bit more care must be taken
when the `loadNpmTasks()` function is called, because Grunt must take care of
the `node_modules` folder in which modules are locally installed. The following
code is the whole implementation of the `loadNpmTasks()` function. I will
explain it in detail.

{% highlight js %}
task.loadNpmTasks = function(name) {
  loadTasksMessage('"' + name + '" local Npm module');
  var root = path.resolve('node_modules');
  var pkgfile = path.join(root, name, 'package.json');
  var pkg = grunt.file.exists(pkgfile) ? grunt.file.readJSON(pkgfile) : {keywords: []};
 
  // Process collection plugins.
  if (pkg.keywords && pkg.keywords.indexOf('gruntcollection') !== -1) {
    loadTaskDepth++;
    Object.keys(pkg.dependencies).forEach(function(depName) {
      // Npm sometimes pulls dependencies out if they're shared, so find
      // upwards if not found locally.
      var filepath = grunt.file.findup('node_modules/' + depName, {
        cwd: path.resolve('node_modules', name),
        nocase: true
      });
      if (filepath) {
        // Load this task plugin recursively.
        task.loadNpmTasks(path.relative(root, filepath));
      }
    });
    loadTaskDepth--;
    return;
  }
 
  // Process task plugins.
  var tasksdir = path.join(root, name, 'tasks');
  if (grunt.file.exists(tasksdir)) {
    loadTasks(tasksdir);
  } else {
    grunt.log.error('Local Npm module "' + name + '" not found. Is it installed?');
  }
};
{% endhighlight %}

The first step is to load the `package.json` file in a directory named
`node_modules/<name>`, relative to the current working directory. If this file
exists, and if it contains the `"gruntcollection"` keyword in its `"keywords"`
section, the package is loaded as a meta-task; otherwise, the
`node_modules/<name>/tasks` folder is scanned and files are loaded as simple
local tasks, as described above.

The case of a meta-task is very interesting. In the first place, because it
exploits the `"dependencies"` section of the `package.json` file and, in the
second place, because it tries to find modules scanning upward to the path of
the meta-module. To better explain how meta-tasks works, it can be useful to
take a look at a typical meta-task’s `package.json` file. Following is the
`package.json` file of the `grunt-contrib` package. Please pay attention to the
`"dependencies"` and `"keywords"` section.

{% highlight json %}
{
  "name": "grunt-contrib",
  "description": "The entire grunt-contrib suite.",
  "version": "0.6.1",
  "homepage": "https://github.com/gruntjs/grunt-contrib",
  "author": {
    "name": "Grunt Team",
    "url": "http://gruntjs.com/"
  },
  "repository": {
    "type": "git",
    "url": "git://github.com/gruntjs/grunt-contrib.git"
  },
  "bugs": {
    "url": "https://github.com/gruntjs/grunt-contrib/issues"
  },
  "licenses": [
    {
      "type": "MIT",
      "url": "https://github.com/gruntjs/grunt-contrib/blob/master/LICENSE-MIT"
    }
  ],
  "engines": {
    "node": ">= 0.8.0"
  },
  "dependencies": {
    "grunt-contrib-clean": "~0.4.0",
    "grunt-contrib-coffee": "~0.6.4",
    "grunt-contrib-compass": "~0.1.3",
    "grunt-contrib-compress": "~0.4.6",
    "grunt-contrib-concat": "~0.1.3",
    "grunt-contrib-connect": "~0.2.0",
    "grunt-contrib-copy": "~0.4.0",
    "grunt-contrib-cssmin": "~0.5.0",
    "grunt-contrib-csslint": "~0.1.1",
    "grunt-contrib-handlebars": "~0.5.8",
    "grunt-contrib-htmlmin": "~0.1.1",
    "grunt-contrib-imagemin": "~0.1.2",
    "grunt-contrib-jade": "~0.5.0",
    "grunt-contrib-jasmine": "~0.4.1",
    "grunt-contrib-jshint": "~0.3.0",
    "grunt-contrib-jst": "~0.5.0",
    "grunt-contrib-less": "~0.5.0",
    "grunt-contrib-livereload": "~0.1.2",
    "grunt-contrib-nodeunit": "~0.1.2",
    "grunt-contrib-qunit": "~0.2.0",
    "grunt-contrib-requirejs": "~0.4.0",
    "grunt-contrib-sass": "~0.2.2",
    "grunt-contrib-stylus": "~0.5.0",
    "grunt-contrib-uglify": "~0.2.0",
    "grunt-contrib-watch": "~0.3.1",
    "grunt-contrib-yuidoc": "~0.4.0"
  },
  "devDependencies": {
    "grunt": "~0.4.1",
    "grunt-contrib-internal": "~0.4.3"
  },
  "peerDependencies": {
    "grunt": "~0.4.0"
  },
  "keywords": [
    "gruntplugin",
    "gruntcollection"
  ]
}
{% endhighlight %}

Reading the `"dependencies"` section is a trivial task once the `package.json`
file is parsed, because it is only an object containing package names as keys
and corresponding versions as values. So, a trivial
`Object.keys(pkg.dependencies)` can return an array of every dependency declared
in the `package.json` file.

To traverse the directory hierarchy Grunt relies on the `findup-sync` module. In
short, `findup-sync` traverses a path upward and scans for a given location,
implementing the same searching mechanism performed by Node when a module is
required. In the case of Grunt, the path to search for is
`node_modules/<depName>` (instead of `node_modules`), and the search must start
from the path of the current module (`path.resolve('node_modules', name)`). If
the `node_modules/<depName>` directory is found, the `loadNpmTasks()` method is
called recursively on that directory and the search process starts again with a
new root path.
