---
layout: post
title:  "The Sling authenticator"
date:   2013-09-13
---

The authentication code in Sling is maybe one of the most complicated part of the whole project. It is a a critic functionality but, at the same time, it provides a lot of flexibility and it is a good example of an extensible API.

The `SlingAuthenticator`, which is the core of the authentication bundle, implements the `AuthenticationSupport` service interface, which is used to authenticate requests to the `SlingMainServlet`, and the `Authenticator` service, which is used to authenticate a request programmatically

It also listens for services with a `sling.auth.requirements` parameter. The parameter contains an array of paths, and each path can be prefixed with `+` or `-` to enable or disable authentication for the specified path. This functionality allows arbitrary services to disable authentication for a given URL (e.g. an authentication form), or to enforce it for a every URL beginning with a specified path.

The `SlingAuthenticator` also listens for `AuthenticationHandler` services. For an authentication handler to be registered, it must expose a path service property. The property contains an array of paths, and each path can be absolute paths without a host name, or an absolute path containing a host name and port. The authentication handler is used only if a request path matches one of these paths.

Finally, the `SlingAuthenticator` listens for `AuthenticationInfoPostProcessor` services. As we will see, these are low-level services used in the authentication workflow.

## Main workflow

The main workflow is executed every time a request comes into the system.

1.If the incoming request has an attribute named `"org.apache.sling.auth.core.ResourceResolver"`, and the value of this attribute is an instance of `ResourceResolver`, the request is already authenticated. No further processing is needed.
1.If the request is not authenticated, perform authentication. The authentication process returns true if the request is correctly authenticated and the framework should continue processing the request. The proper authentication process is described by the authentication workflow.

## Authentication workflow

The authentication workflow is executed when a request has no authentication information attached to it and must be authenticated before being processed by the framework.

1. Extract authentication credentials from the request.
  1. Find all authentication handlers applicable for the request path. This involve matching the path service property of the authentication handlers with the path of the request.
  1. For each authentication handler, invoke the method `extractCredentials()`. If the method returns a non `null` `AuthenticationInfo` object, select the current authentication handler for the authentication process. Invoke the `getFeedbackHandler()` method on the selected authentication handler and save the return value in the `AuthenticationInfo` object.
  1. If no authentication handler is found, and if basic HTTP authentication is not disabled by configuration, invoke the `extractCredentials()` method of `HttpBasicAuthenticationHandler`. If this method returns a non `null` `AuthenticationInfo` object, invoke the `getFeedbackHandler()` method and save the return value into the `AuthenticationInfo` object. Use the basic HTTP authentication to authenticate the request.
  1. If no authentication handler is found, use an `AuthenticationInfo` object representing the anonymous user. The user and password of the anonymous user are read from the configuration.
1. Post-process the authentication credentials.
  1. For each `AuthenticationInfoPostProcessor` service, call the method `postProcess()`. If this method throws a `LoginException`, the authentication workflow stops and a `j_reason` attribute is set on the request. The login workflow starts immediately.
1. Check the `AuthenticationInfo` object returned by one of the authentication handlers.
  1. If the object is a special `DOING_AUTH` instance, then the authentication handler is already in the process of performing the authentication process.
  1. If the object is a special `FAIL_AUTH` instance, then the authentication handler completed the authentication process, but it failed because of invalid credentials. The login workflow starts immediately.
  1. If the authentication info represents an anonymous user, then try to find a resource resolver which is able to authenticate the anonymous user. If such a resource resolver exists, then the authentication process ends successfully; otherwise, it fails with a `LoginException` and the login workflow starts immediately.
  1. Final case: the object has valid credentials. The resource resolver workflow starts immediately.

## Resource resolver workflow

This workflow receives as input the `AuthenticationInfo` object returned by one authentication handler. Other than the usual authentication data, this object contains the following information:

- An optional authentication handler represented by a AuthenticationFeedbackHandler object.
- An optional object which, if not null, will instruct the system to send a post-login event.

The workflow executes the following steps:

1. Handle impersonation. The system allows the request to impersonate another user. The request is inspected to check if impersonation should be enabled.
1. Check if the request has a special cookie. The cookie name can be configured, but it defaults to `sling.sudo`. The value of the cookie will be used as impersonation user.
1. Check if the request has a special parameter. The parameter name can be configured, but it defaults to `sudo`. The value of this parameter will be used as impersonation user. The parameter always overrides the cookie. The parameter can have a special value of `"-"` to disable user impersonation at all.
1. If user impersonation is not disabled and a valid impersonation user is set in the request, this information is set inside the authentication info object.
1. Connect to a resource resolver by using the information contained in the authentication info object.
1. If the authentication info object contains impersonation information, set a cookie in the response. This allows subsequent requests to use the same impersonation user unless it is explicitly changed or disabled. If the authentication info object doesn’t contain impersonation information, the response is configured to remove any impersonation cookie already present in the user agent.
1. A post-login event is sent if the authentication handler returned a valid authentication feedback handler during the authentication workflow.
1. The resource resolver is set as an attribute of the request.
1. If the authentication handler provided a feedback handler, the `authenticationSucceeded()` method of the feedback handler is called. This method can return a `false` value to stop the process to execute further. This is useful if the authentication handler wants to take care of the response for a successful authentication.
1. If the request has a `j_validate` parameter with value `"true"`, send a successful response and don’t process the request any further.
1. Set some attributes on the request: store the remote user (the user which is used to login into the resource resolver) and the authentication type (provided by the authentication handler).

Authenticating to the resource resolver can still throw a `LoginException`, in example if an invalid impersonation user is provided. In this case, a fallback workflow is performed.

1. If the authentication handler provided a feedback handler, call the `authenticationFailed()` method of the feedback handler. If the feedback handler commits a response, the workflow ends, because it assumes that the feedback handler will take care of communicating to the user that authentication fails.
1. If the feedback handler didn’t take care of sending a response to the user, the login workflow starts immediately.

## Login workflow

This workflow is called under different circumstances:

- Post-processing authentication information fails.
- An authentication handler fails to extract credential information from the request.
- The request has no credential information, but no anonymous resource resolver exists.
- No resource resolver exists which is able to authenticate the user, or to use an impersionation user configured in the request.

The workflow executes the following steps:

1. If the request has a `j_validate` parameter with value `"true"`, send a 403 HTTP response and stop the workflow. This step covers the use case when a client only wants to validate that a user name and password combination is valid to login into the system.
1. If the request doesn’t contain the `User-Agent` header, presume that the request is a WEBDav request. If the basic HTTP authenticator is enabled, a 401 HTTP response is sent to signal the client that it is not authorized to access the resource. Otherwise, a 403 HTTP response is sent to signal the client that the server refused to process the request.
1. If the request is AJAX, a 403 http response is sent.
1. The request can be authenticated, so authentication handlers must come into play.
1. Find all authentication handlers applicable for the request path. This involve matching the `path` service property of the authentication handlers with the path of the request.
1. For each authentication handler, invoke the `requestCredentials()` method. If this method returns `true`, the workflow ends because one authentication handler exists to take care of asking the user for login information.
1. If no authentication handler has been found, and if basic HTTP authentication has not been disabled by configuration, invoke the method `requestCredentials()` of `HttpBasicAuthenticationHandler`.
1. If no authentication handler has been found at all, then a `NoAuthenticationHandlerException` is thrown.
