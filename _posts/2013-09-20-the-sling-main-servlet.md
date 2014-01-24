---
layout: post
title:  "The Sling main servlet"
date:   2013-09-20
---

The Sling main servlet is the component which handles incoming requests after authorization is correctly performed. When the Sling main servlet is called, a resource resolver has already been stored into the request. The resource resolver, together with other components and services, will be used to perform the real handling of the request. The most important role of the Sling main servlet is to integrate every required component with the OSGi HTTP Service, and coordinate them to correctly process a request.

The Sling main servlet depends on the following services:

- the error handler, which is called every time an error occurs during the processing of the request;
- the servlet resolver, which finds scripts (wrapped into servlet instances) capable of rendering the requested resource;
- the MIME type service, which takes care of discovering and registering the MIME types understood by the underlying HTTP service;
- the authentication support service, which connects the Sling dynamic authentication framework to the OSGi HTTP Service.

Most of the services listed above can be considered simple utilities, and will not be discussed here. The real work of the Sling main servlet is performed by two other components, which are the core of the request handling process: the Sling HTTP context and the default implementation of the Sling request processor.

The Sling HTTP context, in reality, is a very dumb class. Nevertheless, it is of capital importance for the whole framework, because it creates a connection between Sling and the OSGi HTTP Service. Every time a new servlet instance is registered using the OSGi HTTP Service, an HTTP context must be specified. If not specified, OSGi will create a default one, but Sling is a very good example for when implementing a custom HTTP context is important. The purpose of the OSGi HTTP context is to provide common services to the servlets registered with it: handling of security, reading of resources, handling of MIME types. Sling already implements very well two of these three services (security and MIME types), and it doesn’t use one of them at all (reading resources). The main role of the Sling HTTP context, thus, is to take the MIME types service and the security services implemented in Sling and make them available to underlying HTTP implementation. Why Sling doesn’t use the resource service from the underlying HTTP context? The answer is trivial: because the main role of Sling itself is to resolve resources, and the resource resolution process is what makes Sling a special framework.

The Sling request processor is a different beast. It’s not a glue component, it is the default implementation of the request handling logic of Sling. It is such an important component that the wise Sling developers decided to make it available to the rest of the application as an independent service. But, at the moment, I don’t have a clue of the real usefulness of such a decision. Anyway, the Sling request process performs the following steps for each incoming request:

1. create the Sling request and the Sling response;
1. use the resource resolver, contained in the request, to resolve the current resource;
1. use the servlet resolver (a dependency service of the Sling main servlet) to get a servlet instance which will render the resource;
1. execute request filters, if available;
1. execute component filters, if available;
1. finally, render the resource with the appropriate servlet.

After these steps are executed, the every possible filter is called, the script is executed to render the resource, and the response is correctly sent to the client. In case of error, instead, the request processor executes the error filters defined in the system and uses the error handler (again a dependency service of the Sling main servlet) to render an error response and send it to the client.
