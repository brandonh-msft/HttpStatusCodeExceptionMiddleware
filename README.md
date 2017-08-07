# Using Middleware to trap Exceptions in Asp.Net Core
Recently while working on a project I found myself writing a ton of boilerplate code throughout that looked something like this:
```
private readonly Lazy<JSchema> _newItemSchema = new Lazy<JSchema>(() => Helpers.GetSchema(@"POST-request.json"));
[HttpPost]
public async Task<IActionResult> AddNewItem()
{
    var requestJson = Request.ParseBodyToObject(_newItemSchema.Value);
    if (!requestJson.Success)
    {
#if DEBUG
        return BadRequest(requestJson.Object);
#else
        return BadRequest();
#endif
    }
    
    var newItem = requestJson.Object;
...
```
The idea here is to have a JSON Schema defined for the body for any of the POST methods on my web api. Using this, I can run quick validation on the body to make sure it's good and, if it's not, throw a _meaningful_ HTTP 400 BAD REQUEST back to the caller.

The problem I found, however, was that I couldn't simply "return BadRequest()" from within my `ParseRequestBodyToObjectAsync` helper method. Rather I had to resort to returning a `ValueTuple` containing `Success` (whether or not schema validation passed) and `Object` (either the successfully parsed object, or a JSON response that contained any schema validation error messages).

You can see where this would result in this type of code all over my codebase, for every POST endpoint.

I wanted to clean this up. But how could I get **helper** methods to return an Http response out to the API and stop execution of the caller? Sounds like an exception...

## Asp.Net Core Middleware

Enter middleware. If you've ever looked at the boilerplate for an Asp.Net Core Web Api project, you've seen this code:

```
public void Configure(IApplicationBuilder app, IHostingEnvironment env, ILoggerFactory loggerFactory)
{
    loggerFactory.AddConsole(Configuration.GetSection("Logging"));
    loggerFactory.AddDebug();

    if (env.IsDevelopment())
    {
        app.UseDeveloperExceptionPage();
    }
    else
    {
        app.UseExceptionHandler();
    }

    app.UseStaticFiles();
    app.UseMvc();
}
```
See all those `Use...` calls? That's adding built-in middleware to your web app. And like a good platform, Asp.Net Core enables developers to write their own middleware as well.

### Custom Middleware

To create your own middleware, you simply start with the C# Item Template within Visual Studio. Right click your project and choose Add | New Item...
![](https://brandonhmsdnblog.blob.core.windows.net/images/2017/07/31/6c9f588e.png)

Next, choose the Middleware Class template:
![](https://brandonhmsdnblog.blob.core.windows.net/images/2017/07/31/2017-07-31_14-46-55.png)

Behold, your boilerplate middleware

### How Middleware works
Middleware is added to your app during Startup, as you saw above. **The order in which you call the `Use...` methods *does* matter!** Middleware is "waterfalled" down through until either all have been executed, or one stops execution (in the case of our exception handling, we'll be writing ours so it stops execution. More on that later).

The first things passed to your middleware is a request delegate. This is a delegate that takes the current HttpContext object and executes it. Your middleware saves this off upon creation, and uses it in the `Invoke()` step.
`Invoke()` is where the work is done. Whatever you want to do to the request/response as part of your middleware is done here. Some other usages for middleware might be to authorize a request based on a header or inject a header in to the request or response. For more examples, check out [the Middleware documentation](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/middleware).

## Writing our middleware
For my middleware, I settled on two new constructs:
1. A new exception type - `HttpStatusCodeException` - which I would throw whenever I wanted to stop execution of the current request and send back a response with a specific Status Code and Body. It turned out looking like this:
```
public class HttpStatusCodeException : Exception
{
    public int StatusCode { get; set; }
    public string ContentType { get; set; } = @"text/plain";

    public HttpStatusCodeException(int statusCode)
    {
        this.StatusCode = statusCode;
    }

    public HttpStatusCodeException(int statusCode, string message) : base(message)
    {
        this.StatusCode = statusCode;
    }

    public HttpStatusCodeException(int statusCode, Exception inner) : this(statusCode, inner.ToString()) { }

    public HttpStatusCodeException(int statusCode, JObject errorObject) : this(statusCode, errorObject.ToString())
    {
        this.ContentType = @"application/json";
    }
}
```
2. The middleware handler of the new exception. This middleware would need to trap the exception, set the necessary properties on the response, send it back and, most importantly, not let any more middleware fire off. It ended up looking like this:
```
public class HttpStatusCodeExceptionMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<HttpStatusCodeExceptionMiddleware> _logger;

    public HttpStatusCodeExceptionMiddleware(RequestDelegate next, ILoggerFactory loggerFactory)
    {
        _next = next ?? throw new ArgumentNullException(nameof(next));
        _logger = loggerFactory?.CreateLogger<HttpStatusCodeExceptionMiddleware>() ?? throw new ArgumentNullException(nameof(loggerFactory));
    }

    public async Task Invoke(HttpContext context)
    {
        try
        {
            await _next(context);
        }
        catch (HttpStatusCodeException ex)
        {
            if (context.Response.HasStarted)
            {
                _logger.LogWarning("The response has already started, the http status code middleware will not be executed.");
                throw;
            }

            context.Response.Clear();
            context.Response.StatusCode = ex.StatusCode;
            context.Response.ContentType = ex.ContentType;

            await context.Response.WriteAsync(ex.Message);

            return;
        }
    }
}

// Extension method used to add the middleware to the HTTP request pipeline.
public static class HttpStatusCodeExceptionMiddlewareExtensions
{
    public static IApplicationBuilder UseHttpStatusCodeExceptionMiddleware(this IApplicationBuilder builder)
    {
        return builder.UseMiddleware<HttpStatusCodeExceptionMiddleware>();
    }
}
```

## Using our middleware
Last but not least, we need to tell our app to _use_ the middleware we just wrote. To do this, we simply change our `ConfigureAppliction` method in Startup.cs from this:
```
if (env.IsDevelopment())
{
    app.UseDeveloperExceptionPage();
}
else
{
    app.UseExceptionHandler();
}
```
to this:
```
if (env.IsDevelopment())
{
    app.UseDeveloperExceptionPage();
    app.UseHttpStatusCodeExceptionMiddleware();
}
else
{
    app.UseHttpStatusCodeExceptionMiddleware();
    app.UseExceptionHandler();
}
```
Notice the difference in order when in development mode vs not. This is important as the Developer Exception page passes through the exception to our handler so in order to get the best of both worlds, you want the Developer Page handler first. In production, however, since the default Exception Page halts execution, we definitely to _not_ want that one first.

## The result
My previous boilerplate schema validation code, then, now becomes simply this:
```
private readonly Lazy<JSchema> _newItemSchema = new Lazy<JSchema>(() => _schemas.GetSchema(@"POST-request.json"));
[HttpPost]
public async Task<IActionResult> AddNewItem()
{
    var requestObject = Request.ParseBodyToObject(_newItemSchema.Value);
```
because now inside my `ParseBodyToObject` call I'm simply doing
```
throw new HttpStatusCodeException(StatusCodes.Status400BadRequest, @"You sent bad stuff");
```
which pops all the way up the stack and stops execution of the request within my app.
