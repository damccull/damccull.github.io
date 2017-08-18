Title: ASP.NET Core 2.0 Authentication and Authorization System Demystified
Published: 8/18/2017 02:32
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

<div class="note">NOTE: This post is in draft form, and will be updated regularly. This note will be removed for the final version.</div>

There is a component that exists in ASP.NET Core that conjures up an enchanted shield that protects portions (or all) of your website from unauthorized access. Like many people, I have used this component from the beginning of my journey, but have never understood it. It was conjured up by a wizard and provided a magical barrier between my website and the world. That’s not how it really works, of course, but without the right knowledge, it might as well.

While trying to figure out how to fix an error in my code, I happened to ask the right question at the right time on the right Slack channel.  David Fowler, who happens to be one of the core maintainers of aspnetcore, decided to give everyone present a lesson on how the authentication system (auth system, from now on) works in ASP.NET Core 2.0. This article is based on the information in his impromptu lesson.

Understanding the system first requires understanding its components and behaviors. They can be broken down into verbs, authentication handlers, and middleware. I will cover each of these individually and then demonstrate how they work together in an example auth request. Since ASP.NET Core's most common authentication handler is the Cookies auth handler, these examples will use cookie authentication.

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

Authentication handlers are components that actually implement the behavior of the 5 verbs above. The default auth handler provided by ASP.NET Core is the Cookies authentication handler which implements all 5 of the verbs. It is important to note, however, that a auth handler is not required to implement all of the verbs. Oauth handlers, for instance, do not implement the SignIn verb, but rather pass off that responsibility to another auth handler, such as the Cookies auth handler.

Authentication handlers must be registered with the auth system in order to be used and are associated with schemes. A scheme is just a string that identifies a unique auth handler in a dictionary of auth handlers. The default scheme for the Cookies auth handler is “Cookies”, but it can be changed to anything. Multiple auth handlers can be used side by side, and sometimes (like in the earlier example of the Oauth handlers) use functionality provided by other auth handlers.

## Authentication Middleware

A middleware is a module that can be inserted into the startup sequence and is run on every request. The one this article is concerned with is the authentication middleware. This code checks if a user is authenticated (or not) on every request. Recall that the Authenticate verb gets the user info, but only if it exists. When the request is run, the authentication middleware asks the default scheme auth handler to run its authentication code. The auth handler returns the information to the authentication middleware which then populates the HttpContext.User object with the returned information.

## Authentication and Authorization Flow

All of these components must be used together in the auth system in order to successfully authenticate and authorize a user to access a resource. The process begins with the unauthenticated user sending a request for a resource that requires authorization to access.

Here is an example flow for cookies authentication:

<img src="/images/aspnetcore-auth-system-demystified_auth-flow.svg" />

1. The request arrives at the server.
2. The authentication middleware calls the default handler's Authenticate method and populates the HttpContext.User object with any available information.
3. The request arrives at the controller action.
4. If the action is not decorated with the `[Authorize]` attribute, display the page and stop here.
5. If the action **is** decorated with `[Authorize]`, the auth filter checks if the user was authenticated.
6. If the user was not, the auth handler calls Challenge, redirecting to the appropriate signin authority.
7. Once the signin authority directs the user back to the app, the app checks if the user is authorized to view the page.
8. If the user is authorized, it displays the page, otherwise it calls Forbit, which displays a 'not authorized' page.

## Code Example

When the app first starts it triggers the ConfigureServices() and Configure() methods in the Startup class.

The auth system is interesting and well designed. It's very extensible and will easily work with custom authentication handlers. Understanding how this system works under the hood is the first step in using it beyond the defaults in the templates. All kinds of custom authentication processes are possible by using the components themselves instead of just relying on the templates and convenience methods. Now go write code.
