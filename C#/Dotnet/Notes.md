# Middleware

3 types of delegates: `Run`, `Use`, and `Map`

`Run` delegates terminate the pipeline. Once you hit it the pipeline is terminated. If you don't specify a response, .NET gives back an empty 200 OK. A `Run`  does NOT have a `next` parameter.

`Use` middleware is the classic middleware that allows you to do things. In order to continue the pipeline you need to call `next` . Note that you can still do things after `next` is called (think logging here). If you don't call `next` you short circuit the pipeline and the execution stops.

`Map` delegates _branch_ the pipeline. What does this mean? It means once it's hit, it's executed on the side and the pipeline of middlewares no longer matters.

Within this branching strategy you 2 types of mapping delegates: `MapWhen` and `UseWhen`. `MapWhen` works the same as `Map` with the exception it takes a predicate. `UseWhen` also uses a predicate and is useful because it's *re-joins* the middleware pipeline after executed.
# Middleware Order

Remember middleware execution goes "up and down". When a request comes in it goes through all the middlewares until its handled and then back up through all the same middlewares again. Therefore, with middlewares, ORDER MATTERS!

This behaviour means 2 things:

- The global exception handler middleware needs to be the first registered because (next line)
- If an exception is thrown along the middleware chain it's bubbles up to the exception handler middleware

Other things to take into account:

Static file middleware - this should be called early in the pipeline so that it can short circuit request without going through the pipeline. Also this middleware is **unauthenticated!** Any files served under this middleware are publicly available

Your custom middleware always runs after .NET build in middleware EXCEPT the `Endpoint` middleware (your route handlers essentially)

# Specific Middlewares and caveats

## Cookie Policy middleware

What this middleware does is to check a specific cookie for cookie consent and, if consent is not provided, then all cookies marked as "Non-Essential" will not be sent to the client. By default the name of the cookie is "CookieConsent" and the value is "yes/no". This can however be overridden in config.

## CORS

In terms of configuration is pretty straightforward. The crux here is that you can specify multiple policies (or just have a default one) and you can decide whether to evaluate the CORS policy at 3 different levels: app level, controller level, or endpoint level. Note that this middleware needs to come before **UseResponseCaching**

## Healthchecks

At the most basic level this just creates an endpoint that returns an `OK`. You can configure this is multiple ways such as add/remove headers, allow/deny CORS, do DB healthchecks, etc

## Response Caching

This is a simple way of caching responses. However this uses `IMemoryCache` under the hood which has limitations. Also there doesn't seem to be a way of using this with minimal APIs, you need to add it as a controller attribute. I don't see this being used over redis for example.
 Note: there's a similar concept called `OutputCaching` but that applies only to MVC controllers or Razor pages. That is essentially "cache the html file response";

## Adding or Removing specific headers

One way of adding/removing specific headers is via a middleware that appends before the response is sent back to the client like this:

```C#
app.Use(async (context, next) =>
{
    // Do work that can write to the Response.
    Console.WriteLine("yo from Use 1");

    context.Response.OnStarting(() =>
    {
        context.Response.Headers.Append("size", "big");
        context.Response.Headers.Append("Server", "nginx");
        return Task.CompletedTask;
    });

    await next.Invoke(context);
    // Do logging or other work that doesn't write to the Response.
});
```

This works for some headers but not for others. For example, the `Server` header cannot be removed like this because it's added later in the process by Kestrel or IIS. One thing you can do is fake it like this.

## Session middleware

This middleware adds support for sessions. What it does it to create an in-memory cache and give you methods to set or get the session cookie. It also sends the cookie to the client automatically.

``` C#
builder.Services.AddDistributedMemoryCache();

builder.Services.AddSession(options =>
{
    options.IdleTimeout = TimeSpan.FromMinutes(5);
    options.Cookie.HttpOnly = true;
    options.Cookie.IsEssential = true;
});

(...)

app.MapGet("setsession", (HttpContext context) =>
{
    context.Session.SetString("fruits", "bananas, apples, pears");
    return Results.Ok();
});

app.MapGet("getsession", (HttpContext context) =>
{
    var fruits = context.Session.GetString("fruits");
    return Results.Ok(fruits);
});
```

The only config that is not straightforward is `IdleTimeout`. This is how long a session on the cache (not the cookie) is valid form if it's not accessed. Every time you access the session cache, the timeout resets.

## Static Files Middleware

Gives you a way to serve static files. You need to create a `wwwroot` folder and place files inside it. This is the most basic configuration. A file in the `wwwroot` folder can be reached like this: `http://localhost:5006/images/dog.jpg` (this assumes a folder called `images` exists inside `wwwroot`).

You have another middleware named `UseDefaultFiles` which looks inside the `wwwroot` folder and serves a default file like `index.html`. Note that this needs to come *before* `useStaticFiles`

```C#
app.UseDefaultFiles();

app.UseStaticFiles(new StaticFileOptions()
{
    OnPrepareResponse = (ctx) =>
    {
        ctx.Context.Response.Headers.Append("Cache-Control", "public, max-age=3600");
    }
});
```

## Rate Limiting Middleware

.NET out of the box supports rate limiting. There are 4 algorithms to choose from. You can apply rate limiting globally meaning "one algorithm applies to all requests" OR you can do something called "partitioned rate limiting". Partitioned means you apply different rules to different users (or partitions). A good example of this is if you want to allow users which have an API-key more request than users that do not have an API key. 

``` C#
builder.Services.AddRateLimiter(options =>
{
    options.GlobalLimiter = PartitionedRateLimiter.Create<HttpContext, string>(httpContext =>
    {
        string apiKey = httpContext.Request.Headers["X-API-KEY"].ToString() ?? "no-key";

        switch (apiKey)
        {
            case "premium-key":
                return RateLimitPartition.GetFixedWindowLimiter(apiKey, (pk) =>
                {
                    return new FixedWindowRateLimiterOptions() { PermitLimit = 10, Window = TimeSpan.FromSeconds(30) };
                });
            default:
                return RateLimitPartition.GetFixedWindowLimiter(apiKey, (pk) =>
                {
                    return new FixedWindowRateLimiterOptions() { PermitLimit = 1, Window = TimeSpan.FromSeconds(30) };
                });
        }
    });

(...)

app.UseRateLimiter();
```

# Filters

Filters allow you to perform cross concern actions. Think of this for things such as checking if a user is allowed to do something, changing the arguments that are passed to a controller, adding or removing things to the responses that are sent to clients or handling exceptions

They have their own pipeline that runs after the middleware pipeline.

![[Pasted image 20250413161132.png]]

Filters can be used at 3 different levels:
1. At a controller ACTION level (meaning if affects only this action)
2. At a controller level (meaning if affects all actions in a controller)
3. At a global level (meaning it affects every action in every controller)

What happens when you have multiple filters for a given pipeline? The order above is preserved, you go from most general (global) to most specific (controller action filter).

Also note there's an interface you can implement to override filter order - `IOrderedFilter`
## Filters Vs Filter Attributes

They achieve the same thing, i.e.: filter stuff, but their usage is different. Filters are to be registered with the DI container whereas attributes are to used in controllers as decorators.

## Filters and DI

Filters can be added by type or by instance. If it's added by instance, it's a singleton and you only get one instance. If it's a type filter, then an instance is created for every request and any constructor dependencies are added via DI. **Filters that are implemented as attributes and added directly to controllers cannot have DI dependencies.**

## Service Filters Attributes

Service Filters Attributes are filters that are added via DI. This is to solve the problem of attribute filters not allowing any dependencies. How to use? Like this: `[ServiceFilter(typeof(MyCustomFilter))]`

How do you tell .NET a filter is meant to be used via DI or via attributes? Depends on what you inherit from:

```C#
public class MyValidationFilter : Attribute, IActionFilter //extends from Attribute so attribute filter
{}
```

```C#
public class MyValidationFilter : IActionFilter //doesn't extend so ServiceFilter
{}
```

Are Service Filters singletons? It depends on how you register them in the DI container. If you want them to be a singleton you can.
## TypeFilter Attributes

This is similar to Service filters with the exception that these filters are resolved on the fly and so you don't have to register them in the DI container.  Also you can pass constructor arguments to them, something you cannot do with Service Filters. How to use them? Like this: `[TypeFilter(typeof(MyAuditFilter), Arguments = new object[] { "AdminAction" })]`

## Type Filters Vs Service Filters - When to use which?

In terms of performance they are more or less the same. They are not the same in terms of reusability. Because you can pass constructor arguments to TypeFilters you risk sprinkling logic all over the place  by passing different parameters. Is this a problem? Well 🤷‍♂️.


# that Note about the filter where you cannot infer the T 