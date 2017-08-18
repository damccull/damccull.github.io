Title: ASP.NET Core 2.0 Authentication and Authorization System Demystified
Published: 8/17/2017
Tags:
    - ASP.Net Core
    - Lesson
---
<style>
  .note {
    margin-bottom:25px;
    margin-left:auto;
    margin-right:auto;
    border:1px solid lightblue;
    border-radius: 10px;
    background-color: #e0e0ff;
    padding: 5px;
    font-size:.8em;
    font-family: sans-serif;
    width:85%;
  }
</style>

There is a component that exists in ASP.NET Core that conjures up an enchanted shield that protects portions (or all) of your website from unauthorized access. Like many people, I have used this component from the beginning of my journey, but have never understood it. It was conjured up by a wizard and provided a magical barrier between my website and the world. That’s not how it really works, of course, but without the right knowledge, it might as well.

While trying to figure out how to fix an error in my code, I happened to ask the right question at the right time on the right Slack channel.  David Fowl, who happens to be one of the core maintainers of aspnetcore, decided to give everyone present a lesson on how the authentication and authorization system (auth system, from now on) works in ASP.NET Core 2.0. This article is based on the information in his impromptu lesson.

Understanding the system first requires understanding its components and behaviors. They can be broken down into verbs, authentication handlers, and middleware. I will cover each of these individually and then demonstrate how they work together in an example auth request.

## Verbs

There are 5 verbs (these can also be thought of as commands or behaviors) that are invoked by the auth system, and are not necessarily called in order. These are all independent actions that do not communicate among themselves, however, when used together allow users to sign in and access pages otherwise denied. Here is a very brief description of what each verb is responsible for. We’ll go into more depth further in the article.


<div class="note">Note: These are behaviors, not methods (although there are methods of the same names implementing these behaviors).</div>

* *Authenticate*
  * Gets the user’s information if any exists (e.g. decoding the user’s cookie, if one exists)
* *Challenge*
  * Requests authentication by the user (e.g. showing a login page)
* *SignIn*
  * Persists the user’s information somewhere (e.g. writes a cookies)
* *SignOut*
  * Removes the user’s persisted information (e.g. deletes the cookies)
* *Forbid*
  * Denies access to a resource for unauthenticated users or authenticated but unauthorized users (e.g. displaying a “not authorized” page)

## Authentication Handlers

Authentication handlers are components that actually implement the behavior of the 5 verbs above. The default handler provided by ASP.NET Core is the Cookies authentication handler which implements all 5 of the verbs. It is important to note, however, that a handler is not required to implement all of the verbs. Oauth handlers, for instance, do not implement the SignIn verb, but rather pass off that responsibility to another handler, such as the Cookies handler.

Handlers must be registered with the auth system in order to be used and are associated with schemes. A scheme is just a string that identifies a unique handler in a dictionary of handlers. The default scheme for the Cookies handler is “Cookies”, but it can be changed to anything. Multiple handlers can be used side by side, and sometimes (like in the earlier example of the Oauth handlers) use functionality provided by other handlers.

## Authentication Middleware

A middleware is a module that can be inserted into the startup sequence and is run on every request. The one this article is concerned with is the authentication middleware. This code checks if a user is authenticated (or not) on every request. Recall that the Authenticate verb gets the user info, but only if it exists. When the request is run, the authentication middleware asks the default scheme handler to run its authentication code, and then populates the HttpContext.User object with the returned information.

## Authentication and Authorization Flow

All of these components must be used together in the auth system in order to successfully authenticate and authorize a user to access a resource. The process begins with the unauthenticated user sending a request for a resource that requires authorization to access.

<img src="/images/aspnetcore-auth-system-demystified_auth-flow.svg" />

1. The request arrives at the server.
2. The authentication middleware calls the default handler's Authenticate method and populates the HttpContext.User object with any available information.
3. The request arrives at the controller action.
4. If the action is not decorated with the [Authorize] attribute, display the page and stop here.
5. The handler checks if the user was authenticated by the middleware.
6. If the user was not, the handler calls Challenge, redirecting to the appropriate signin authority.
7. Once the signin authority directs the user back to the app, the app checks if the user is authorized to view the page.
8. If the user is authorized, it displays the page, otherwise it displays a 'not authorized' page.
