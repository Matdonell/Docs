---
title: Routing
author: rick-anderson
ms.author: riande
manager: wpickett
ms.date: 10/14/2016
ms.topic: article
ms.assetid: bbbcf9e4-3c4c-4f50-b91e-175fe9cae4e2
ms.prod: aspnet-core
uid: fundamentals/routing
---
# Routing

By [Ryan Nowak](https://github.com/rynowak), [Steve Smith](http://ardalis.com), and [Rick Anderson](https://twitter.com/RickAndMSFT)

Routing is used to map requests to route handlers. Routes are configured when the application starts up, and can extract values from the URL that will be used for request processing. Routing functionality is also responsible for generating links using the defined routes in ASP.NET apps.

This document covers the low level ASP.NET Core routing. For ASP.NET Core MVC routing, see [Routing to Controller Actions](../mvc/controllers/routing.md)

[View or download sample code](https://github.com/aspnet/Docs/tree/master/aspnetcore/fundamentals/routing/sample)

## Routing basics

Routing uses *routes* (implementations of [`IRouter`](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Routing/IRouter/index.html#Microsoft.AspNetCore.Routing.IRouter)) to:

* map incoming requests to *route handlers*

* generate URLs used in responses

Generally an app has a single collection of routes. The route collection is processed in order. Requests look for a match in the route collection by . Responses use routing to generate URLs.

Routing is connected to the [middleware](middleware.md) pipeline by the [`RouterMiddleware`](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Builder/RouterMiddleware/index.html#Microsoft.AspNetCore.Builder.RouterMiddleware) class. [ASP.NET MVC](../mvc/overview.md) adds routing to the middleware pipeline as part of its configuration. To learn about using routing as a standalone component, see [using-routing-middleware](#using-routing-middleware).

<a name=url-matching-ref></a>

### URL matching

URL matching is the process by which routing dispatches an incoming request to a *handler*. This process is generally based on data in the URL path, but can be extended to consider any data in the request. The ability to dispatch requests to separate handlers is key to scaling the size and complexity of an application.

Incoming requests enter the [`RouterMiddleware`](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Builder/RouterMiddleware/index.html#Microsoft.AspNetCore.Builder.RouterMiddleware) which calls the [`RouteAsync`](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Routing/IRouter/index.html#Microsoft.AspNetCore.Routing.IRouter.RouteAsync) method on each route in sequence. The [`IRouter`](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Routing/IRouter/index.html#Microsoft.AspNetCore.Routing.IRouter) instance chooses whether to *handle* the request by setting the [`RouteContext`](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Routing/RouteContext/index.html#Microsoft.AspNetCore.Routing.RouteContext) [`Handler`](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Routing/RouteContext/index.html#Microsoft.AspNetCore.Routing.RouteContext.Handler) to a non-null
[`RequestDelegate`](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Http/RequestDelegate/index.html#Microsoft.AspNetCore.Http.RequestDelegate). If a handler is set a route, it will be invoked to process the request and no further routes will be processed. If all routes are executed, and no handler is found for a request, the middleware calls *next* and the next middleware in the request pipeline is invoked.

The primary input to `RouteAsync` is the [`RouteContext`](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Routing/RouteContext/index.html#Microsoft.AspNetCore.Routing.RouteContext) [`HttpContext`](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Routing/RouteContext/index.html#Microsoft.AspNetCore.Routing.RouteContext.HttpContext) associated with the current request. The `RouteContext.Handler` and [`RouteContext`](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Routing/RouteContext/index.html#Microsoft.AspNetCore.Routing.RouteContext) [`RouteData`](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Routing/RouteContext/index.html#Microsoft.AspNetCore.Routing.RouteContext.RouteData) are outputs that will be set after a successful match.

A successful match during `RouteAsync` also will set the properties of the `RouteContext.RouteData` to appropriate values based on the request processing that was done. The `RouteContext.RouteData` contains important state information about the *result* of a route when it successfully matches a request.

[`RouteData`](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Routing/RouteData/index.html#Microsoft.AspNetCore.Routing.RouteData) [`Values`](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Routing/RouteData/index.html#Microsoft.AspNetCore.Routing.RouteData.Values) is a dictionary of *route values* produced from the route. These values are usually determined by tokenizing the URL, and can be used to accept user input, or to make further dispatching decisions inside the application.

[`RouteData`](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Routing/RouteData/index.html#Microsoft.AspNetCore.Routing.RouteData) [`DataTokens`](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Routing/RouteData/index.html#Microsoft.AspNetCore.Routing.RouteData.DataTokens)  is a property bag of additional data related to the matched route. `DataTokens` are provided to support associating state data with each route so the application can make decisions later based on which route matched. These values are developer-defined and do **not** affect the behavior of routing in any way. Additionally, values stashed in data tokens can be of any type, in contrast to route values which must be easily convertable to and from strings.

[`RouteData`](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Routing/RouteData/index.html#Microsoft.AspNetCore.Routing.RouteData) [`Routers`](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Routing/RouteData/index.html#Microsoft.AspNetCore.Routing.RouteData.Routers) is a list of the routes that took part in successfully matching the request. Routes can be nested inside one another, and the `Routers` property reflects the path through the logical tree of routes that resulted in a match. Generally the first item in `Routers` is the route collection, and should be used for URL generation. The last item in `Routers` is the route that matched.

### URL generation

URL generation is the process by which routing can create a URL path based on a set of route values. This allows for a logical separation between your handlers and the URLs that access them.

URL generation follows a similar iterative process, but starts with user or framework code calling into the [`GetVirtualPath`](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Routing/IRouter/index.html#Microsoft.AspNetCore.Routing.IRouter.GetVirtualPath) method of the route collection. Each *route* will then have its `GetVirtualPath` method called in sequence until a non-null [`VirtualPathData`](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Routing/VirtualPathData/index.html#Microsoft.AspNetCore.Routing.VirtualPathData) is returned.

The primary inputs to `GetVirtualPath` are:

* [`VirtualPathContext`](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Routing/VirtualPathContext/index.html#Microsoft.AspNetCore.Routing.VirtualPathContext) [`HttpContext`](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Routing/VirtualPathContext/index.html#Microsoft.AspNetCore.Routing.VirtualPathContext.HttpContext)

* [`VirtualPathContext`](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Routing/VirtualPathContext/index.html#Microsoft.AspNetCore.Routing.VirtualPathContext) [`Values`](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Routing/VirtualPathContext/index.html#Microsoft.AspNetCore.Routing.VirtualPathContext.Values)

* [`VirtualPathContext`](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Routing/VirtualPathContext/index.html#Microsoft.AspNetCore.Routing.VirtualPathContext) [`AmbientValues`](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Routing/VirtualPathContext/index.html#Microsoft.AspNetCore.Routing.VirtualPathContext.AmbientValues)

Routes primarily use the route values provided by the `Values` and `AmbientValues` to decide where it is possible to generate a URL and what values to include. The `AmbientValues` are the set of route values that were produced from matching the current request with the routing system. In contrast, `Values` are the route values that specify how to generate the desired URL for the current operation. The `HttpContext` is provided in case a route needs to get services or additional data associated with the current context.

>[!TIP]
> Think of `Values` as being a set of overrides for the `AmbientValues`. URL generation tries to reuse route values from the current request to make it easy to generate URLs for links using the same route or route values.

The output of `GetVirtualPath` is a [`VirtualPathData`](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Routing/VirtualPathData/index.html#Microsoft.AspNetCore.Routing.VirtualPathData). `VirtualPathData` is a parallel of `RouteData`; it contains the `VirtualPath` for the output URL as well as the some additional properties that should be set by the route.

The [`VirtualPathData`](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Routing/VirtualPathData/index.html#Microsoft.AspNetCore.Routing.VirtualPathData) [`VirtualPath`](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Routing/VirtualPathData/index.html#Microsoft.AspNetCore.Routing.VirtualPathData.VirtualPath) property contains the *virtual path* produced by the route. Depending on your needs you may need to process the path further. For instance, if you want to render the generated URL in HTML you need to prepend the base path of the application.

The [`VirtualPathData`](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Routing/VirtualPathData/index.html#Microsoft.AspNetCore.Routing.VirtualPathData) [`Router`](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Routing/VirtualPathData/index.html#Microsoft.AspNetCore.Routing.VirtualPathData.Router) is a reference to the route that successfully generated the URL.

The [`VirtualPathData`](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Routing/VirtualPathData/index.html#Microsoft.AspNetCore.Routing.VirtualPathData) [`DataTokens`](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Routing/VirtualPathData/index.html#Microsoft.AspNetCore.Routing.VirtualPathData.DataTokens) properties is a dictionary of additional data related to the route that generated the URL. This is the parallel of `RouteData.DataTokens`.

### Creating routes

Routing provides the [`Route`](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Routing/Route/index.html#Microsoft.AspNetCore.Routing.Route) class as the standard implementation of `IRouter`. `Route` uses the *route template* syntax to define patterns that will match against the URL path when [`RouteAsync`](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Routing/IRouter/index.html#Microsoft.AspNetCore.Routing.IRouter.RouteAsync) is called. `Route` will use the same route template to generate a URL when [`GetVirtualPath`](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Routing/IRouter/index.html#Microsoft.AspNetCore.Routing.IRouter.GetVirtualPath) is called.

Most applications will create routes by calling `MapRoute` or one of the similar extension methods defined on [`IRouteBuilder`](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Routing/IRouteBuilder/index.html#Microsoft.AspNetCore.Routing.IRouteBuilder). All of these methods will create an instance of `Route` and add it to the route collection.

> [!NOTE]
> [`MapRoute`](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Builder/MapRouteRouteBuilderExtensions/index.html#Microsoft.AspNetCore.Builder.MapRouteRouteBuilderExtensions.MapRoute) doesn't take a route handler parameter - it only adds routes that will be handled by the [`DefaultHandler`](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Routing/IRouteBuilder/index.html#Microsoft.AspNetCore.Routing.IRouteBuilder.DefaultHandler). Since the default handler is an [`IRouter`](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Routing/IRouter/index.html#Microsoft.AspNetCore.Routing.IRouter), it may decide not to handle the request. For example, ASP.NET MVC is typically configured as a default handler that only handles requests that match an available controller and action. To learn more about routing to MVC, see [ð"§ Routing to Controller Actions](../mvc/controllers/routing.md).

This is an example of a `MapRoute` call used by a typical ASP.NET MVC route definition:

````csharp
routes.MapRoute(
    name: "default",
    template: "{controller=Home}/{action=Index}/{id?}");
````

This template will match a URL path like `/Products/Details/17` and extract the route values `{ controller = Products, action = Details, id = 17 }`. The route values are determined by splitting the URL path into segments, and matching each segment with the *route parameter* name in the route template. Route parameters are named. They are defined by enclosing the parameter name in braces `{ }`.

The template above could also match the URL path `/` and would produce the values `{ controller = Home, action = Index }`. This happens because the `{controller}` and `{action}` route parameters have default values, and the `id` route parameter is optional. An equals `=` sign followed by a value after the route parameter name defines a default value for the parameter. A question mark `?` after the route parameter name defines the parameter as optional. Route parameters with a default value *always* produce a route value when the route matches - optional parameters will not produce a route value if there was no corresponding URL path segment.

See [route-template-reference](#route-template-reference) for a thorough description of route template features and syntax.

This example includes a *route constraint*:

````csharp
routes.MapRoute(
    name: "default",
    template: "{controller=Home}/{action=Index}/{id:int}");
````

This template will match a URL path like `/Products/Details/17`, but not `/Products/Details/Apples`. The route parameter definition `{id:int}` defines a *route constraint* for the `id` route parameter. Route constraints implement `IRouteConstraint` and inspect route values to verify them. In this example the route value `id` must be convertable to an integer. See [route-constraint-reference](#route-constraint-reference) for a more detailed explaination of route constraints that are provided by the framework.

Additional overloads of `MapRoute` accept values for `constraints`, `dataTokens`, and `defaults`. These additional parameters of `MapRoute` are defined as type `object`. The typical usage of these parameters is to pass an anonymously typed object, where the property names of the anonymous type match route parameter names.

The following two examples create equivalent routes:

````csharp
routes.MapRoute(
    name: "default_route",
    template: "{controller}/{action}/{id?}",
    defaults: new { controller = "Home", action = "Index" });

routes.MapRoute(
    name: "default_route",
    template: "{controller=Home}/{action=Index}/{id?}");
````

>[!TIP]
> The inline syntax for defining constraints and defaults can be more convenient for simple routes. However, there are features such as data tokens which are not supported by inline syntax.

This example demonstrates a few more features:

````csharp
routes.MapRoute(
  name: "blog",
  template: "Blog/{*article}",
  defaults: new { controller = "Blog", action = "ReadArticle" });
````

This template will match a URL path like `/Blog/All-About-Routing/Introduction` and will extract the values `{ controller = Blog, action = ReadArticle, article = All-About-Routing/Introduction }`. The default route values for `controller` and `action` are produced by the route even though there are no corresponding route parameters in the template. Default values can be specified in the route template. The `article` route parameter is defined as a *catch-all* by the appearance of an asterix `*` before the route parameter name. Catch-all route parameters capture the remainder of the URL path, and can also match the empty string.

This example adds route constraints and data tokens:

````csharp
routes.MapRoute(
    name: "us_english_products",
    template: "en-US/Products/{id}",
    defaults: new { controller = "Products", action = "Details" },
    constraints: new { id = new IntRouteConstraint() },
    dataTokens: new { locale = "en-US" });
````

This template will match a URL path like `/en-US/Products/5` and will extract the values `{ controller = Products, action = Details, id = 5 }` and the data tokens `{ locale = en-US }`.

![image](routing/_static/tokens.png)

<a name=id1></a>

### URL generation

The `Route` class can also perform URL generation by combining a set of route values with its route template. This is logically the reverse process of matching the URL path.

>[!TIP]
> To better understand URL generation, imagine what URL you want to generate and then think about how a route template would match that URL. What values would be produced? This is the rough equivalent of how URL generation works in the `Route` class.

This example uses a basic ASP.NET MVC style route:

````csharp
routes.MapRoute(
    name: "default",
    template: "{controller=Home}/{action=Index}/{id?}");
````

With the route values `{ controller = Products, action = List }`, this route will generate the URL `/Products/List`. The route values are substituted for the corresponding route parameters to form the URL path. Since `id` is an optional route parameter, it's no problem that it doesn't have a value.

With the route values `{ controller = Home, action = Index }`, this route will generate the URL `/`. The route values that were provided match the default values so the segments corresponding to those values can be safely omitted. Note that both URLs generated would round-trip with this route definition and produce the same route values that were used to generate the URL.

>[!TIP]
> An app using ASP.NET MVC should use [`UrlHelper`](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Mvc/Routing/UrlHelper/index.html#Microsoft.AspNetCore.Mvc.Routing.UrlHelper) to generate URLs instead of calling into routing directly.

For more details about the URL generation process, see [url-generation-reference](#url-generation-reference).

<a name=using-routing-middleware></a>

## Using Routing Middleware

To use routing middleware, add it to the **dependencies** in *project.json*:

`"Microsoft.AspNetCore.Routing": <current version>`

Add routing to the service container in *Startup.cs*:

[!code-csharp[Main](../fundamentals/routing/sample/RoutingSample/Startup.cs?highlight=3&start=11&end=14)]

Routes must configured in the `Configure` method in the `Startup` class. The sample below uses these APIs:

* [`RouteBuilder`](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Routing/RouteBuilder/index.html#Microsoft.AspNetCore.Routing.RouteBuilder)
* [`Build`](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Routing/RouteBuilder/index.html#Microsoft.AspNetCore.Routing.RouteBuilder.Build)
* [`MapGet`](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Routing/RequestDelegateRouteBuilderExtensions/index.html#Microsoft.AspNetCore.Routing.RequestDelegateRouteBuilderExtensions.MapGet)  Matches only HTTP GET requests
* [`UseRouter`](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Builder/RoutingBuilderExtensions/index.html#Microsoft.AspNetCore.Builder.RoutingBuilderExtensions.UseRouter)

<!-- literal_block {"xml:space": "preserve", "source": "fundamentals/routing/sample/RoutingSample/Startup.cs", "ids": [], "linenos": false, "highlight_args": {"linenostart": 1}} -->

````
public void Configure(IApplicationBuilder app, ILoggerFactory loggerFactory)
{
    var trackPackageRouteHandler = new RouteHandler(context =>
    {
        var routeValues = context.GetRouteData().Values;
        return context.Response.WriteAsync(
            $"Hello! Route values: {string.Join(", ", routeValues)}");
    });

    var routeBuilder = new RouteBuilder(app, trackPackageRouteHandler);

    routeBuilder.MapRoute(
        "Track Package Route",
        "package/{operation:regex(^track|create|detonate$)}/{id:int}");

    routeBuilder.MapGet("hello/{name}", context =>
    {
        var name = context.GetRouteValue("name");
        // This is the route handler when HTTP GET "hello/<anything>"  matches
        // To match HTTP GET "hello/<anything>/<anything>, 
        // use routeBuilder.MapGet("hello/{*name}"
        return context.Response.WriteAsync($"Hi, {name}!");
    });            

    var routes = routeBuilder.Build();
    app.UseRouter(routes);
````

The table below shows the responses with the given URIs.

| URI | Response  |
| ------- | -------- |
| /package/create/3  | Hello! Route values: [operation, create], [id, 3] |
| /package/track/-3  | Hello! Route values: [operation, track], [id, -3] |
| /package/track/-3/ | Hello! Route values: [operation, track], [id, -3]  |
| /package/track/ | \<Fall through, no match> |
| GET /hello/Joe | Hi, Joe! |
| POST /hello/Joe | \<Fall through, matches HTTP GET only> |
| GET /hello/Joe/Smith | \<Fall through, no match> |

If you are configuring a single route, call `app.UseRouter` passing in an `IRouter` instance. You won't need to call `RouteBuilder`.

The framework provides a set of extension methods for creating routes such as:

* [`MapRoute`](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Builder/MapRouteRouteBuilderExtensions/index.html#Microsoft.AspNetCore.Builder.MapRouteRouteBuilderExtensions.MapRoute)
* [`MapGet`](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Routing/RequestDelegateRouteBuilderExtensions/index.html#Microsoft.AspNetCore.Routing.RequestDelegateRouteBuilderExtensions.MapGet)
* [`MapPost`](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Routing/RequestDelegateRouteBuilderExtensions/index.html#Microsoft.AspNetCore.Routing.RequestDelegateRouteBuilderExtensions.MapPost)
* [`MapPut`](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Routing/RequestDelegateRouteBuilderExtensions/index.html#Microsoft.AspNetCore.Routing.RequestDelegateRouteBuilderExtensions.MapPut)
* [`MapDelete`](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Routing/RequestDelegateRouteBuilderExtensions/index.html#Microsoft.AspNetCore.Routing.RequestDelegateRouteBuilderExtensions.MapDelete)
* [`MapVerb`](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Routing/RequestDelegateRouteBuilderExtensions/index.html#Microsoft.AspNetCore.Routing.RequestDelegateRouteBuilderExtensions.MapVerb)

Some of these methods such as `MapGet` require a [`RequestDelegate`](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Http/RequestDelegate/index.html#Microsoft.AspNetCore.Http.RequestDelegate) to be provided. The `RequestDelegate` will be used as the *route handler* when the route matches. Other methods in this family allow configuring a middleware pipeline which will be used as the route handler. If the *Map* method doesn't accept a handler, such as `MapRoute`, then it will use the [`DefaultHandler`](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Routing/IRouteBuilder/index.html#Microsoft.AspNetCore.Routing.IRouteBuilder.DefaultHandler).

The `Map[Verb]` methods use constraints to limit the route to the HTTP Verb in the method name. For example, see [MapGet](https://github.com/aspnet/Routing/blob/1.0.0/src/Microsoft.AspNetCore.Routing/RequestDelegateRouteBuilderExtensions.cs#L85-L88) and [MapVerb](https://github.com/aspnet/Routing/blob/1.0.0/src/Microsoft.AspNetCore.Routing/RequestDelegateRouteBuilderExtensions.cs#L156-L180).

<a name=route-template-reference></a>

## Route Template Reference

Tokens within curly braces (`{ }`) define *route parameters* which will be bound if the route is matched. You can define more than one route parameter in a route segment, but they must be separated by a literal value. For example `{controller=Home}{action=Index}` would not be a valid route, since there is no literal value between `{controller}` and `{action}`. These route parameters must have a name, and may have additional attributes specified.

Literal text other than route parameters (for example, `{id}`) and the path separator `/` must match the text in the URL. Text matching is case-insensitive and based on the decoded representation of the URLs path. To match the literal route parameter delimiter `{` or  `}`, escape it by repeating the character (`{{` or `}}`).

URL patterns that attempt to capture a filename with an optional file extension have additional considerations. For example, using the template `files/{filename}.{ext?}` - When both `filename` and `ext` exist, both values will be populated. If only `filename` exists in the URL, the route matches because the trailing period `.` is  optional. The following URLs would match this route:

* `/files/myFile.txt`
* `/files/myFile.`
* `/files/myFile`

You can use the `*` character as a prefix to a route parameter to bind to the rest of the URI - this is called a *catch-all* parameter. For example, `blog/{*slug}` would match any URI that started with `/blog` and had any value following it (which would be assigned to the `slug` route value). Catch-all parameters can also match the empty string.

Route parameters may have *default values*, designated by specifying the default after the parameter name, separated by an `=`. For example, `{controller=Home}` would define `Home` as the default value for `controller`. The default value is used if no value is present in the URL for the parameter. In addition to default values, route parameters may be optional (specified by appending a `?` to the end of the parameter name, as in `id?`). The difference between optional and "has default" is that a route parameter with a default value always produces a value; an optional parameter has a value only when one is provided.

Route parameters may also have constraints, which must match the route value bound from the URL. Adding a colon `:` and constraint name after the route parameter name specifies an *inline constraint* on a route parameter. If the constraint requires arguments those are provided enclosed in parentheses `( )` after the constraint name. Multiple inline constraints can be specified by appending another colon `:` and constraint name. The constraint name is passed to the [`IInlineConstraintResolver`](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Routing/IInlineConstraintResolver/index.html#Microsoft.AspNetCore.Routing.IInlineConstraintResolver) service to create an instance of [`IRouteConstraint`](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Routing/IRouteConstraint/index.html#Microsoft.AspNetCore.Routing.IRouteConstraint) to use in URL processing. For example, the route template `blog/{article:minlength(10)}` specifies the
`minlength` constraint with the argument `10`. For more description route constraints, and a listing of the constraints provided by the framework, see [route-constraint-reference](#route-constraint-reference).

The following table demonstrates some route templates and their behavior.

| Route Template | Example Matching URL | Notes |
| -------- | -------- | ------- |
| hello  | /hello  | Only matches the single path `/hello` |
| {Page=Home} | / | Matches and sets `Page` to `Home` |
| {Page=Home}  | /Contact  \ Matches and sets `Page` to `Contact` |
| {controller}/{action}/{id?} | /Products/List | Maps to `Products` controller and `List`  action |
| {controller}/{action}/{id?} | /Products/Details/123  |  Maps to `Products` controller and  `Details` action.  `id` set to 123 |
| {controller=Home}/{action=Index}/{id?} | /  |  Maps to `Home` controller and `Index`  method; `id` is ignored. |

Using a template is generally the simplest approach to routing. Constraints and defaults can also be specified outside the route template.

>[!TIP]
> Enable [Logging](logging.md) to see how the built in routing implementations, such as `Route`, match requests.

<a name=route-constraint-reference></a>

## Route Constraint Reference

Route constraints execute when a `Route` has matched the syntax of the incoming URL and tokenized the URL path into route values. Route constraints generally inspect the route value associated via the route template and make a simple yes/no decision about whether or not the value is acceptable. Some route constraints use data outside the route value to consider whether the request can be routed. For example, the [`HttpMethodRouteConstraint`](https://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Routing/Constraints/HttpMethodRouteConstraint/index.html#httpmethodrouteconstraint-class) can accept or reject a request based on its HTTP verb.

>[!WARNING]
> Avoid using constraints for **input validation**, because doing so means that invalid input will result in a 404 (Not Found) instead of a 400 with an appropriate error message. Route constraints should be used to **disambiguate** between similar routes, not to validate the inputs for a particular route.

The following table demonstrates some route constraints and their expected behavior.

| constraint | Example | Example Match | Notes |
| --------   | ------- | ------------- | ----- |
| `int` | {id:int} | 123 | Matches any integer |
| `bool`  | {active:bool} | true | Matches `true` or `false` |
| `datetime` | {dob:datetime} | 2016-01-01  | Matches a valid `DateTime` value (in the invariant culture - see options) |
| `decimal` | {price:decimal} | 49.99 | Matches a valid `decimal` value |
| `double`  | {weight:double} | 4.234 | Matches a valid `double` value |
| `float`  | {weight:float} | 3.14 | Matches a valid `float` value |
| `guid`  | {id:guid} | CD2C1638-1638-72D5-1638-DEADBEEF1638 | Matches a valid `Guid` value |
| `long` | {ticks:long} | 123456789 | Matches a valid `long` value |
| `minlength(value)` | {username:minlength(4)} | Rick | String must be at least 4 characters |
| `maxlength(value)` | {filename:maxlength(8)} | somefile | String must be no more than 8 characters |
| `length(min,max)` | {filename:length(4,16)} | *Somefile.txt* | String must be at least 8 and no more than 16 characters long |
| `min(value)` | {age:min(18)} | 19 | Value must be at least 18 |
| `max(value)` | {age:max(120)} |  91 | Value must be no more than 120 |
| `range(min,max)` | {age:range(18,120)} | 91 | Value must be at least 18 but no more than 120 |
| `alpha` | {name:alpha} | Rick | String must consist of alphabetical characters |
| `regex(expression)` | {ssn:regex(^d{3}-d{2}-d{4}$)} | 123-45-6789 | String must match the regular expression |
| `required`  | {name:required} | Rick |  Used to enforce that a non-parameter value is present during URL generation |

>[!WARNING]
> Route constraints that verify the URL can be converted to a CLR type (such as `int` or `DateTime`) always use the invariant culture - they assume the URL is non-localizable. The framework-provided route constraints do not modify the values stored in route values. All route values parsed from the URL will be stored as strings. For example, the [Float route constraint](https://github.com/aspnet/Routing/blob/1.0.0/src/Microsoft.AspNetCore.Routing/Constraints/FloatRouteConstraint.cs#L44-L60) will attempt to convert the route value to a float, but the converted value is used only to verify it can be converted to a float.

>[!TIP]
> To constrain a parameter to a known set of possible values, you can use a regular expression ( for example `{action:regex(list|get|create)}`. This would only match the `action` route value to `list`, `get`, or `create`. If passed into the constraints dictionary, the string "list|get|create" would be equivalent. Constraints that are passed in the constraints dictionary (not inline within a template) that don't match one of the known constraints are also treated as regular expressions.

<a name=url-generation-reference></a>

## URL Generation Reference

The example below shows how to generate a link to a route given a dictionary of route values and a `RouteCollection`.

[!code-csharp[Main](../fundamentals/routing/sample/RoutingSample/Startup.cs?range=45-59)]

The `VirtualPath` generated at the end of the sample above is `/package/create/123`.

The second parameter to the [`VirtualPathContext`](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Routing/VirtualPathContext/index.html#Microsoft.AspNetCore.Routing.VirtualPathContext) constructor is a collection of *ambient values*. Ambient values provide convenience by limiting the number of values a developer must specify within a certain request context. The current route values of the current request are considered ambient values for link generation. For example, in an ASP.NET MVC app if you are in the `About` action of the `HomeController`, you don't need to specify the controller route value to link to the `Index` action (the ambient value of `Home` will be used).

Ambient values that don't match a parameter are ignored, and ambient values are also ignored when an explicitly-provided value overrides it, going from left to right in the URL.

Values that are explicitly provided but which don't match anything are added to the query string. The following table shows the result when using the route template `{controller}/{action}/{id?}`.

| Ambient Values | Explicit Values | Result |
| -------------   | -------------- | ------ |
| controller="Home" | action="About" | `/Home/About` |
| controller="Home" | controller="Order",action="About" | `/Order/About` |
| controller="Home",color="Red" | action="About" | `/Home/About` |
| controller="Home" | action="About",color="Red" | `/Home/About?color=Red`

If a route has a default value that doesn't correspond to a parameter and that value is explicitly provided, it must match the default value. For example:

````csharp
routes.MapRoute("blog_route", "blog/{*slug}",
  defaults: new { controller = "Blog", action = "ReadPost" });
````

Link generation would only generate a link for this route when the matching values for controller and action are provided.
