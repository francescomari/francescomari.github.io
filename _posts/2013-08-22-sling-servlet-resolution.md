---
layout: post
title:  "Sling servlet resolution"
date:   2013-08-22
---

This process starts when Sling receives an HTTP request and it correctly finds a resource for the request. When this process starts, Sling already has the following information:

- The request path, comprised of resource path, extension and selectors.
- The resource type, identified by the `sling:resourceType` prooperty of the resource.
- The resource supertype, identified by the `sling:resourceSuperType` property of the resource.
- The resource type hierarchy

With this pieces of information, Sling is able to build the resource type hierarchy, which is the first step in resolving the correct servlet. A typical resource type hierarchy for a resource `R` is something like

```
T -> ST(1) -> ST(2) -> ... -> ST(n) -> DT
```

where `T` is the `sling:resourceType` of `R` and `DT` is the default resource type. In the case of Sling, the default resource type is always `sling/servlet/default`. `ST(1)` is the `sling:resourceSuperType` of `R`, if it is specified, or the `sling:resourceSuperType` of `T`. Each supertype `ST(x)`, where x is greater than or equal to 2, is found by following the `sling:resourceSuperType` of `ST(x-1)`.

The value of `sling:resourceType` or `sling:resourceSuperType` is a string describing a relative or an absolute path. A relative path can be expressed like `blog/page` or `blog:page`; an absolute path can only be expressed like `/apps/blog/page`. When an absolute path is used, that resource is selected and no further logic is executed. When a relative path is used, the Sling resource resolver prepends the values of the script path array to it to obtain an absolute path. In example, if the resource type is `blog/page` and the script path array is `[/apps, /libs]`, the possible absolute paths are `/apps/blog/page` and `/libs/blog/page`. The first resolvable absolute path is used as the value of `sling:resourceType` or `sling:resourceSuperType`.

## Searching for scripts

For each element in the type hierarchy, which resolves to a directory in the repository, Sling tries to identify scripts which will be used to rendere the resource. The process is complex, because it checks for a lot of combinations of selectors, extension, resource type name and HTTP method.

In general, this is what Sling checks for each resource type when every single check is executed:

```
selector(i) OR selector(i) + extension              (1)(3)
prefix OR prefix + extension                        (1)
extension                                           (1)
selector(i)                                         (1)(2)(3)
prefix                                              (1)(2)
selector(i) OR selector(i) + extension + method     (3)
prefix OR prefix + extension + method
extension + method
selector(i) OR selector(i) + method                 (2)(3)
prefix OR prefix + method                           (2)
prefix OR prefix + method
method
 
(1) Only if the request method is GET
(2) Only if the request path ends with the default extension (.html)
(3) Only if the request path contains selectors, or if selector(i) is not empty
```

This list will be checked multiple times for each resource type, because Sling will traverse the subdirectories of the resource type which match the selectors contained in the request path. If the request path contains no selectors, Sling will only analyze the content of the resource type, and will not traverse any subdirectory.

Since checking the list is an iterative process, some tokens used in the list will assume different values in each iteration. prefix is the name of the resource type in the first iteration, or the name of the previous matched selector for iterations successive to the second. `selector(i)` is the `i`-th selector specified in the request path, or is empty in the last iteration.

Let’s analyze a quick example to understand how this iterative process works. I assume that Sling received a request for a resource of type `blog/page` with two selectors, `print` and `a4`. The list, in this case, will be checked three times: one for the resource type and one for each selector. The values assumed by `i`, `prefix` and `selector(i)` in each iteration are:

```
i           = 0
prefix      = page
selector(i) = print
 
i           = 1
prefix      = print
selector(i) = a4
 
i           = 2
prefix      = a4
selector(i) = <empty>
```

For each iteration, Sling will go through the list above, stopping at the first match. If a match occurs, the script will be added to a list of candidates with a specific weight according to the specificity of the match. In example, three iterations of the list can produce a maximum of three candidate scripts.

Remember that the search process applies for each resource type found in the hierarchy. So, if a resource has a type hierarchy composed of four resource types and if two selectors are used in the request path, the maximum number of candidate scripts the search process will find is 4 * 3 = 12, where 4 is the number of resource types in the hierarchy and 3 is the number of iterations per resource type.

## Weight

The weight given to each script depends in the first place to the number of selectors that were used when searching for it. In other words, the more Sling has to iterate through the list of possible matches, the more selectors are used to find the script.

If two or more scripts use the same amount of selectors, then they are compared using a score. This score is assigned to each combination in the list of matches. The following table describes the number of selectors used for and the score assigned to each script path in the list of matches:

```
Match                                               Selectors     Score
 
selector(i) OR selector(i) + extension              i + 1         2
prefix OR prefix + extension                        i             3
extension                                           i             2
selector(i)                                         i             0
prefix                                              i + 1         0
selector(i) OR selector(i) + extension + method     i + 1         2
prefix OR prefix + extension + method               i             3
extension + method                                  i             2
selector(i) OR selector(i) + method                 i             0
prefix OR prefix + method                           i + 1         0
prefix OR prefix + method                           i + 1         0
method                                              i             0
```

## The winner

At this point of the resolution process, Sling has an ordered list of script candidates. They are ordered by number of selectors, then by score. But Sling still doesn’t know if the resources represeting these script candidates are usable as scripts.

The very last step in the servlet resolution process is to iterate the list of script candidates, and try to adapt each `Resource` into a servlet. The first candidate which can be adapted to a servlet will be selected as the winner of the resolution process.

Easy, isn’t it?
