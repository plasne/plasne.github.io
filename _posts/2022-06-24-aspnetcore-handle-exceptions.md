---
layout: post
title: Handle Exceptions Middleware in AspNetCore
---

In AspNetCore, it may be advantageous to have the first piece of custom middleware catch any unhandled exceptions and return something proper to the client. This shows an example...

```c#
using System;
using System.IO;
using System.Threading.Tasks;
using Microsoft.AspNetCore.Http;
using Microsoft.Extensions.Logging;

public class HandleExceptionsMiddleware
{
    private readonly RequestDelegate next;
    private readonly ILogger<HandleExceptionsMiddleware> logger;

    public HandleExceptionsMiddleware(RequestDelegate next, ILogger<HandleExceptionsMiddleware> logger)
    {
        this.next = next;
        this.logger = logger;
    }

    public async Task Invoke(HttpContext context)
    {
        var exist = context.Response.Body;
        using var buffer = new MemoryStream();
        context.Response.Body = buffer;
        try
        {
            await this.next(context);
            buffer.Seek(0, SeekOrigin.Begin);
            await buffer.CopyToAsync(exist);
            context.Response.Body = exist;
        }
        catch (HttpStatusException ex)
        {
            context.Response.Body = exist;
            context.Response.Clear();
            this.logger.LogError(ex, "the following exception was raised by a controller...");
            context.Response.StatusCode = ex.StatusCode;
            context.Response.Headers["Content-Type"] = "text/plain";
            await context.Response.WriteAsync(ex.Message);
            await context.Response.CompleteAsync();
        }
        catch (Exception ex)
        {
            context.Response.Body = exist;
            context.Response.Clear();
            this.logger.LogError(ex, "the following exception was raised by a controller...");
            context.Response.StatusCode = 500;
            context.Response.Headers["Content-Type"] = "text/plain";
            await context.Response.WriteAsync("An error occurred while processing your request.");
            await context.Response.CompleteAsync();
        }
    }
}
```

You might register it with an extension like this...

```c#
public static void UseCustomMiddleware(this IApplicationBuilder app)
{
    app.UseMiddleware<HandleExceptionsMiddleware>();
    // more middleware
}
```

It is important that this be the first middleware as it is going to handle the exceptions for all other middleware components and operations.

You can define an HttpStatusException that allows controllers or middleware to throw exceptions that inform the response, like this...

```c#
using System;
using System.Diagnostics.CodeAnalysis;
using System.Net;

public class HttpStatusException : Exception
{
    public HttpStatusException(HttpStatusCode statusCode, string message, Exception innerException = null)
    : base(message, innerException)
    {
        this.StatusCode = (int)statusCode;
    }

    public HttpStatusException(int statusCode, string message, Exception innerException = null)
    : base(message, innerException)
    {
        this.StatusCode = statusCode;
    }

    public int StatusCode { get; }
}
```

Notice that the middleware changes context.Response.Body to use a memory buffer and then puts it back. Using a buffer allows other middleware components to throw exceptions after something has been written to context.Response.Body (for example, a controller writes the output but then a middleware component throws an exception). Otherwise, you can get an exception since the response has already been partially sent to the client.
